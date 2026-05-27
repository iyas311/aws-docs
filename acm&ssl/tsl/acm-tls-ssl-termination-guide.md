# AWS ACM TLS/SSL Termination — Complete Guide

> **AWS Certificate Manager (ACM)** + TLS/SSL termination scenarios using the AWS Console.  
> Covers theory, architecture patterns, and step-by-step setup for each scenario.

---

## Table of Contents

1. [Core Concepts & Theory](#1-core-concepts--theory)
2. [What is ACM?](#2-what-is-acm)
3. [TLS/SSL Termination Types](#3-tlsssl-termination-types)
4. [Scenario 1 — ALB with ACM (Edge Termination)](#4-scenario-1--alb-with-acm-edge-termination)
5. [Scenario 2 — API Gateway with ACM Custom Domain](#5-scenario-2--api-gateway-with-acm-custom-domain)
6. [Scenario 3 — NLB with TLS Passthrough](#6-scenario-3--nlb-with-tls-passthrough)
7. [Scenario 4 — NLB with TLS Termination (ACM)](#7-scenario-4--nlb-with-tls-termination-acm)
8. [Scenario 5 — End-to-End Encryption (Re-Encryption)](#8-scenario-5--end-to-end-encryption-re-encryption)
9. [Scenario 6 — ECS / Fargate with ALB](#9-scenario-6--ecs--fargate-with-alb)
10. [Scenario 7 — Mutual TLS (mTLS) with ACM Private CA](#10-scenario-7--mutual-tls-mtls-with-acm-private-ca)
11. [Scenario 8 — Elastic Beanstalk with ACM](#11-scenario-8--elastic-beanstalk-with-acm)
12. [Scenario 9 — App Runner with ACM Custom Domain](#12-scenario-9--app-runner-with-acm-custom-domain)
13. [ALB Advanced: SNI & Multiple Certificates](#13-alb-advanced-sni--multiple-certificates)
14. [ALB Advanced: Listener Rules & Path-Based Routing](#14-alb-advanced-listener-rules--path-based-routing)
15. [Importing External Certificates into ACM](#15-importing-external-certificates-into-acm)
16. [ACM Private CA — Deep Dive](#16-acm-private-ca--deep-dive)
17. [AWS WAF Integration with ALB TLS](#17-aws-waf-integration-with-alb-tls)
18. [Certificate Validation Methods](#18-certificate-validation-methods)
19. [Security Best Practices](#19-security-best-practices)
20. [Troubleshooting Common Issues](#20-troubleshooting-common-issues)
21. [ACM Limits & Pricing Reference](#21-acm-limits--pricing-reference)
22. [Quick Reference Comparison Table](#22-quick-reference-comparison-table)

---

## 1. Core Concepts & Theory

### What is TLS/SSL?

**Transport Layer Security (TLS)** — and its predecessor SSL (Secure Sockets Layer) — are cryptographic protocols that provide:

- **Confidentiality**: Data is encrypted in transit; no eavesdropping.
- **Integrity**: Data cannot be tampered with mid-transit.
- **Authentication**: Server proves its identity via a digital certificate.

TLS operates by performing a **handshake** between client and server:

1. Client sends `ClientHello` (supported cipher suites, TLS version).
2. Server responds with `ServerHello` + its **certificate**.
3. Client verifies the certificate against a trusted Certificate Authority (CA).
4. Both sides derive a shared session key (via key exchange — e.g., ECDHE).
5. Encrypted communication begins.

### What is TLS Termination?

**TLS termination** (also called SSL offloading) is the process of **decrypting TLS-encrypted traffic at a specific point** in the network path, so downstream services can receive plain HTTP.

```
[Client] ──HTTPS──► [Termination Point] ──HTTP──► [Backend]
                         (decrypt here)
```

The termination point holds the **private key** and **certificate**. It handles all cryptographic work, offloading that burden from backend servers.

### Why Terminate TLS at a Load Balancer?

- **Performance**: CPU-intensive crypto is done once, at the edge.
- **Centralized cert management**: One cert, not one-per-server.
- **Visibility**: Load balancer can inspect HTTP headers, path, cookies for routing.
- **Simplified backends**: Backend servers don't need SSL configuration.

### TLS Termination vs. Passthrough vs. Re-encryption

| Mode | Description | Backend Traffic | Use Case |
|---|---|---|---|
| **Termination** | Decrypt at LB, forward plain HTTP | Unencrypted | Most web apps |
| **Passthrough** | Forward encrypted TCP, backend decrypts | Encrypted (untouched) | Strict compliance, mTLS to backend |
| **Re-encryption** | Decrypt at LB, re-encrypt to backend | New TLS session | End-to-end encryption with LB visibility |

---

## 2. What is ACM?

**AWS Certificate Manager (ACM)** is a managed service that provisions, manages, and renews TLS/SSL certificates for use with AWS services.

### Key Features

- **Free public certificates** for use with integrated AWS services (ALB, NLB, API Gateway, etc.).
- **Automatic renewal**: ACM renews certificates before expiry (no manual intervention).
- **Private CA support**: Issue private certificates via ACM Private CA.
- **Regional service**: Certificates are tied to the AWS region where they are issued.
- **No key export**: Private keys for public ACM certs never leave AWS; they are managed by ACM.

### How ACM Integrates with AWS Services

ACM certificates can be attached to:

- Application Load Balancer (ALB)
- Network Load Balancer (NLB) — TLS listeners
- Amazon API Gateway
- AWS Elastic Beanstalk
- AWS App Runner

> **Important**: ACM certificates **cannot** be installed on EC2 instances directly. For EC2, you must use a third-party cert (e.g., Let's Encrypt).

### Certificate Types in ACM

| Type | Cost | Renewal | Use With |
|---|---|---|---|
| Public ACM Certificate | Free | Automatic | ALB, NLB, API GW |
| Imported Certificate | Free (cert cost external) | Manual | Any AWS service |
| ACM Private CA Certificate | Paid | Automatic | Internal services, mTLS |

---

## 3. TLS/SSL Termination Types

### 3.1 Edge Termination (Most Common)

```
Internet ──HTTPS──► ALB (terminates TLS) ──HTTP──► EC2/ECS/Lambda
```

The ALB holds the ACM certificate and decrypts traffic. Backend receives plain HTTP on port 80 (or any port). Simple, scalable, most widely used.

### 3.2 Passthrough (L4)

```
Internet ──HTTPS──► NLB (TCP passthrough) ──HTTPS──► Backend (holds cert)
```

NLB operates at Layer 4 (TCP). It does not inspect or modify packets. The backend server holds the certificate and performs decryption. Needed when the backend must handle the TLS session itself (e.g., mutual TLS to the app server).

### 3.3 Re-encryption (End-to-End)

```
Internet ──HTTPS──► ALB (terminates, re-encrypts) ──HTTPS──► Backend
```

ALB terminates client TLS, then opens a new TLS connection to the backend. Useful for compliance requirements (data must be encrypted at all hops) while still allowing ALB-level routing/inspection.

---

## 4. Scenario 1 — ALB with ACM (Edge Termination)

### Architecture

```
[Browser] ──HTTPS:443──► [ALB + ACM Cert] ──HTTP:80──► [EC2 / ECS / Lambda Target Group]
```

This is the **most common** pattern. The ALB terminates TLS and routes HTTP to backends.

### Step-by-Step Setup via AWS Console

#### Step 1: Request an ACM Certificate

1. Go to **AWS Console → Certificate Manager → Request a certificate**.
2. Select **Request a public certificate** → **Next**.
3. Enter your domain(s):
   - `example.com`
   - `www.example.com`
   - Or wildcard: `*.example.com`
4. Choose **Validation method**:
   - **DNS validation** (recommended) — add a CNAME record.
   - **Email validation** — approval email to domain contacts.
5. Click **Request**.
6. Complete validation (see Section 12). Status must become **Issued**.

#### Step 2: Create a Target Group

1. Go to **EC2 → Target Groups → Create target group**.
2. **Target type**: Instances (or IP / Lambda).
3. **Protocol**: HTTP, **Port**: 80.
4. **Health check path**: `/health` or `/`.
5. Register your EC2 instances → **Create target group**.

#### Step 3: Create an Application Load Balancer

1. Go to **EC2 → Load Balancers → Create Load Balancer**.
2. Select **Application Load Balancer**.
3. **Name**: `my-alb`
4. **Scheme**: Internet-facing (or Internal for private).
5. **IP address type**: IPv4 (or dualstack).
6. **Network mapping**: Select your VPC + at least 2 Availability Zones + public subnets.
7. **Security groups**: Attach a SG that allows inbound **443** (and optionally 80).

#### Step 4: Add HTTPS Listener

1. Under **Listeners and routing**:
   - Click **Add listener**.
   - **Protocol**: HTTPS, **Port**: 443.
   - **Default action**: Forward to your target group.
2. Under **Secure listener settings**:
   - **Security policy**: `ELBSecurityPolicy-TLS13-1-2-2021-06` (recommended — TLS 1.2+ with TLS 1.3 support).
   - **Default SSL/TLS certificate**: Choose **From ACM** → select your certificate.
3. Optionally add an HTTP (port 80) listener with a **Redirect to HTTPS** action.
4. Click **Create load balancer**.

#### Step 5: Update DNS

1. Copy the ALB **DNS name** (e.g., `my-alb-123456.us-east-1.elb.amazonaws.com`).
2. In Route 53 (or your DNS provider):
   - Create an **A record (Alias)** pointing `example.com` → the ALB DNS name.
   - Create a **CNAME** for `www.example.com` → ALB DNS name.

#### Result

- Client hits `https://example.com` → ALB terminates TLS → forwards HTTP to backend.
- Certificate auto-renews. Zero manual cert management.

---

## 5. Scenario 2 — API Gateway with ACM Custom Domain

### Architecture

```
[Client] ──HTTPS──► [API Gateway Custom Domain + ACM] ──► [Lambda / HTTP Backend]
```

API Gateway provides a subdomain by default (`xyz.execute-api.region.amazonaws.com`). A custom domain gives you `api.example.com`.

### Step-by-Step Setup

#### Step 1: Request ACM Certificate

- For **Regional API** (REST/HTTP): request cert in the same region as your API.
- For **Edge-optimized API**: request cert in **us-east-1** (routes via AWS edge network).

#### Step 2: Create the API Gateway Custom Domain

1. Go to **API Gateway → Custom domain names → Create**.
2. **Domain name**: `api.example.com`.
3. **TLS minimum version**: TLS 1.2.
4. **Endpoint type**:
   - **Regional** — for low-latency single-region APIs.
   - **Edge optimized** — routes via the AWS edge network (cert must be in `us-east-1`).
5. **ACM certificate**: Select the issued certificate.
6. Click **Create domain name**.

#### Step 3: Map the Custom Domain to Your API

1. In the custom domain, go to **API mappings → Configure API mappings**.
2. Select your **API** and **Stage** (e.g., `prod`).
3. **Path**: Leave blank for root, or `/v1` for versioned routes.
4. Save.

#### Step 4: Update DNS

Copy the **API Gateway domain name** (shown as "API Gateway domain name" in the custom domain page) and create a CNAME or Route 53 Alias record for `api.example.com`.

---

## 6. Scenario 3 — NLB with TLS Passthrough

### Architecture

```
[Client] ──TLS:443──► [NLB TCP listener] ──TCP:443──► [EC2 with self-managed cert]
```

NLB passes the encrypted TCP stream directly. The backend EC2 instance holds the certificate and decrypts. Use when:

- Backend needs to see the raw TLS session (e.g., mTLS).
- Compliance requires that the cloud load balancer never sees plaintext.
- Backend already manages its own certificates (e.g., Nginx with Let's Encrypt).

### Step-by-Step Setup

#### Step 1: Create a Target Group

1. **EC2 → Target Groups → Create**.
2. **Target type**: Instances.
3. **Protocol**: **TCP**, **Port**: 443.
4. **Health check**: TCP (or HTTPS if you want the NLB to do a TLS health check).
5. Register instances.

#### Step 2: Create a Network Load Balancer

1. **EC2 → Load Balancers → Create → Network Load Balancer**.
2. **Name**: `my-nlb-passthrough`.
3. **Scheme**: Internet-facing.
4. **IP address type**: IPv4.
5. **Network mapping**: VPC + subnets.

#### Step 3: Add a TCP Listener (Not TLS)

1. **Protocol**: **TCP**, **Port**: 443.
2. **Default action**: Forward to the TCP:443 target group.

> Key: Using TCP (not TLS) at the listener level means NLB does **not** terminate TLS. It blindly forwards the TCP stream.

3. No certificate is needed in this scenario.
4. Click **Create**.

#### Step 4: Configure Backend

Ensure your EC2 instances run a TLS-capable server (Nginx/Apache/HAProxy) with a valid certificate installed directly. The client TLS handshake is completed end-to-end with the EC2 instance.

---

## 7. Scenario 4 — NLB with TLS Termination (ACM)

### Architecture

```
[Client] ──TLS:443──► [NLB TLS listener + ACM cert] ──TCP:80──► [EC2]
```

NLB terminates TLS using an ACM certificate. Unlike ALB, NLB:

- Operates at Layer 4 (no HTTP routing).
- Preserves the client's source IP natively.
- Supports ultra-high throughput with static IPs.
- Does **not** inspect HTTP headers.

### Step-by-Step Setup

#### Step 1: Request ACM Certificate (same region as NLB)

Follow the standard ACM request process (Scenario 1, Step 1).

#### Step 2: Create Target Group

1. **Protocol**: **TCP**, **Port**: 80 (or whatever the backend listens on).
2. Register instances.

#### Step 3: Create NLB with TLS Listener

1. **EC2 → Load Balancers → Create → Network Load Balancer**.
2. Set up name, scheme, subnets.
3. Under **Listeners**:
   - **Protocol**: **TLS**, **Port**: 443.
   - **Default action**: Forward to TCP target group.
4. Under **Secure listener settings**:
   - **Security policy**: `ELBSecurityPolicy-TLS13-1-2-2021-06`.
   - **Default SSL/TLS certificate**: From ACM → select your cert.
5. Create the NLB.

### When to Choose NLB-TLS over ALB-HTTPS

| Feature | ALB HTTPS | NLB TLS |
|---|---|---|
| L7 routing (path/header) | Yes | No |
| Static IP | No (use Global Accelerator) | Yes |
| Source IP preservation | No (X-Forwarded-For header) | Yes (natively) |
| WebSockets / gRPC | Yes | Yes |
| Ultra-low latency | Good | Better |
| Throughput | High | Very High |

---

## 8. Scenario 5 — End-to-End Encryption (Re-Encryption)

### Architecture

```
[Client] ──HTTPS──► [ALB (ACM cert)] ──HTTPS──► [Backend EC2 (self-managed cert)]
```

The ALB terminates the client TLS session, then opens a **new** TLS connection to the backend. Both hops are encrypted. The ALB can still inspect and route based on HTTP attributes.

### Step-by-Step Setup

#### Step 1: Set Up Backend with HTTPS

1. Install a certificate on your EC2 instances (use Let's Encrypt, ACM Private CA, or a self-signed cert for internal-only communication).
2. Configure your web server (Nginx/Apache) to listen on **HTTPS:443** (or a custom HTTPS port).

#### Step 2: Create Target Group (HTTPS Protocol)

1. **EC2 → Target Groups → Create**.
2. **Protocol**: **HTTPS**, **Port**: 443.
3. **Health check protocol**: HTTPS, **path**: `/health`.
4. Under **Health check advanced settings**:
   - If backend uses self-signed certs: disable cert verification (for dev/test).
   - For production: use ACM Private CA–issued certs and enable verification.
5. Register instances.

#### Step 3: Create ALB with HTTPS Listener

1. Same as Scenario 1, Steps 3–4.
2. The HTTPS listener forwards to the **HTTPS target group**.

The ALB now re-encrypts traffic before forwarding to backends. Data is encrypted at both legs of the journey.

---

## 9. Scenario 6 — ECS / Fargate with ALB

### Architecture

```
[Client] ──HTTPS──► [ALB + ACM cert] ──HTTP:8080──► [ECS Fargate Task]
```

Containers in ECS/Fargate don't manage certificates. TLS is terminated at the ALB.

### Step-by-Step Setup

#### Step 1: Create an ECS Fargate Service

1. Create a task definition with your container. Container port: **8080** (or whichever port your app listens on).
2. Create an ECS Cluster (Fargate launch type).
3. Create a Service — leave the load balancer section for now.

#### Step 2: Create Target Group

1. **Target type**: **IP** (required for Fargate — tasks get IPs, not instance IDs).
2. **Protocol**: HTTP, **Port**: 8080.
3. Do not register targets manually — ECS will manage this automatically.

#### Step 3: Create ALB + HTTPS Listener

1. Follow Scenario 1, Steps 3–4.
2. Default rule forwards to the IP-based target group.

#### Step 4: Attach ALB to ECS Service

1. Go to your ECS Service → **Update service**.
2. Under **Load balancing**: Add the ALB.
3. **Container to load balance**: select your container + port 8080.
4. **Target group**: Select the IP-based group created above.
5. Update the service. ECS will automatically register/deregister task IPs as tasks scale.

---

## 10. Scenario 7 — Mutual TLS (mTLS) with ACM Private CA

### What is mTLS?

Standard TLS authenticates only the **server** to the client. **Mutual TLS (mTLS)** requires **both sides** to present certificates — the server authenticates the client too. This is essential for:

- Zero-trust service-to-service communication.
- B2B API security (partners must present a known client cert).
- Regulatory environments (PCI-DSS, HIPAA) requiring strong client identity.

### Architecture

```
[Client with client cert] ──mTLS──► [ALB (mTLS mode + ACM Private CA)] ──HTTP──► [Backend]
```

ALB verifies the client certificate against a **Trust Store** (a bundle of trusted CA certificates). If verification passes, ALB forwards the request and optionally injects client cert details as HTTP headers.

### Step-by-Step Setup

#### Step 1: Create an ACM Private CA

1. Go to **AWS Console → Certificate Manager → Private CA → Create a private CA**.
2. **CA type**: Root CA (or Subordinate if you have an existing PKI hierarchy).
3. **Subject distinguished name**:
   - Common name: `My Internal CA`
   - Organization, Country, etc. as needed.
4. **Key algorithm**: RSA 2048 (or EC P256 for smaller keys).
5. **CA validity**: 10 years (typical for a root CA).
6. Click **Create CA** → then **Activate CA**.
7. Note the **CA ARN** — you'll need it to issue client certificates.

#### Step 2: Issue Client Certificates

1. Go to **Certificate Manager → Request a certificate → Request a private certificate**.
2. Select your **Private CA** from the dropdown.
3. **Domain name**: Use the client's identifier (e.g., `client1.internal.example.com`).
4. **Validity**: 1 year (recommended for client certs).
5. Click **Request**.
6. Download the certificate and private key — distribute to the client application.

#### Step 3: Create a Trust Store in ALB

1. Go to **EC2 → Load Balancers → Trust stores → Create trust store**.
2. **Name**: `my-mtls-truststore`.
3. **CA certificates bundle**: Upload a `.pem` file containing the root CA certificate(s) you trust (export from ACM Private CA: **Actions → Export certificate body**).
4. Click **Create trust store**.

#### Step 4: Enable mTLS on the ALB Listener

1. Select your ALB → **Listeners** tab → select the HTTPS:443 listener → **Edit**.
2. Under **Mutual authentication (mTLS)**:
   - **Mode**: `verify` (client cert is required and verified) or `passthrough` (cert is forwarded but not verified).
   - **Trust store**: Select the trust store created above.
   - **Client certificate handling**: Choose whether to pass cert headers to the backend.
3. Save.

#### Step 5: Access Client Cert Details in Backend

When mTLS is enabled, ALB injects these HTTP headers:

| Header | Content |
|---|---|
| `X-Amzn-Mtls-Clientcert` | Base64-encoded DER certificate |
| `X-Amzn-Mtls-Clientcert-Serial-Number` | Certificate serial number |
| `X-Amzn-Mtls-Clientcert-Issuer` | Issuer DN |
| `X-Amzn-Mtls-Clientcert-Subject` | Subject DN |
| `X-Amzn-Mtls-Clientcert-Validity` | Validity dates |
| `X-Amzn-Mtls-Clientcert-Leaf` | Leaf cert in URL-encoded PEM |

Your backend can read these headers to enforce fine-grained authorization (e.g., only allow requests from `client1.internal.example.com`).

---

## 11. Scenario 8 — Elastic Beanstalk with ACM

### Architecture

```
[Browser] ──HTTPS:443──► [Elastic Beanstalk ALB + ACM cert] ──HTTP──► [EB EC2 instances]
```

Elastic Beanstalk manages the underlying infrastructure. You attach an ACM cert through the EB environment configuration — no need to touch the ALB directly.

### Step-by-Step Setup

#### Step 1: Request ACM Certificate

Same as Scenario 1, Step 1. Ensure it is in the same region as your Beanstalk environment.

#### Step 2: Configure HTTPS in Elastic Beanstalk Console

1. Go to **Elastic Beanstalk → your environment → Configuration**.
2. Under **Load balancer** → click **Edit**.
3. Under **Listeners**, click **Add listener**:
   - **Port**: 443
   - **Protocol**: HTTPS
   - **SSL certificate**: Select your ACM certificate from the dropdown.
   - **SSL policy**: `ELBSecurityPolicy-TLS13-1-2-2021-06`.
4. Optionally change the existing HTTP:80 listener to redirect to HTTPS:
   - Edit the port 80 listener → **Default process**: Select the redirect option.
5. Click **Apply** (this triggers a rolling environment update).

#### Step 3: Update DNS

Point your custom domain to the Beanstalk environment's CNAME (shown in the environment dashboard), using a CNAME record or Route 53 Alias.

#### Notes on EB Load Balancer Types

| EB LB Type | TLS Support | Notes |
|---|---|---|
| Application (ALB) | Full HTTPS + mTLS | Recommended for most apps |
| Network (NLB) | TLS listener | Best for TCP/UDP or static IPs |
| Classic (CLB) | HTTPS | Legacy — avoid for new environments |

---

## 12. Scenario 9 — App Runner with ACM Custom Domain

### Architecture

```
[Browser] ──HTTPS──► [App Runner (built-in TLS)] ──► [Container / Lambda Function]
```

AWS App Runner manages TLS automatically — every service gets a default HTTPS endpoint. Adding a custom domain associates your ACM certificate with that domain.

### Step-by-Step Setup

#### Step 1: Create/Deploy App Runner Service

1. Go to **App Runner → Create service**.
2. Source: ECR image or GitHub repo.
3. Configure port, CPU, memory.
4. Deploy — App Runner auto-provisions a `*.awsapprunner.com` HTTPS endpoint.

#### Step 2: Associate Custom Domain

1. In your App Runner service → **Custom domains** tab → **Add domain**.
2. Enter your domain: `app.example.com`.
3. App Runner provides **CNAME records** to add to your DNS:
   - One CNAME for domain ownership validation.
   - One CNAME to route traffic to App Runner.
4. Add these records in Route 53 or your DNS provider.
5. Status becomes **Active** once DNS propagates.

#### Key Notes

- App Runner **automatically provisions and renews** the ACM certificate for the custom domain — you do not request it manually.
- TLS 1.2 and 1.3 are supported by default.
- No load balancer configuration needed — App Runner handles it all.

---

## 13. ALB Advanced: SNI & Multiple Certificates

### What is SNI?

**Server Name Indication (SNI)** is a TLS extension where the client includes the target hostname in the `ClientHello` message — before the TLS handshake completes. This allows a single ALB listener (port 443) to serve **multiple domains with different certificates**, without needing a separate IP per domain.

```
[Client → api.example.com]   ──SNI: api.example.com──►  |          |  ──► API cert
[Client → app.example.com]   ──SNI: app.example.com──►  |  ALB:443 |  ──► App cert
[Client → admin.example.com] ──SNI: admin.example.com►  |          |  ──► Admin cert
```

All three clients hit the same ALB listener. ALB selects the appropriate ACM certificate based on the SNI hostname.

### Adding Multiple Certificates to an ALB Listener

1. Go to **EC2 → Load Balancers → select your ALB → Listeners tab**.
2. Click on the **HTTPS:443** listener → **Manage certificates**.
3. Under **Add certificates**:
   - Search for certificates by domain name.
   - Select additional ACM certificates.
   - Click **Add**.
4. One certificate is the **default** (used when no SNI match is found). All others are selected via SNI automatically.

> **Limit**: Up to **25 certificates** per ALB listener (plus the default). Use wildcard certs to cover multiple subdomains with a single cert and stay within this limit.

### Certificate Selection Priority

ALB selects the certificate in this order:
1. Exact domain match via SNI (e.g., `api.example.com`).
2. Wildcard match (e.g., `*.example.com`).
3. Default certificate (fallback).

---

## 14. ALB Advanced: Listener Rules & Path-Based Routing

ALB listener rules let you route HTTPS traffic to **different target groups** based on conditions — all on a single HTTPS:443 listener.

### Common Routing Conditions

| Condition | Example | Use Case |
|---|---|---|
| **Path pattern** | `/api/*` | Route API calls to a microservice |
| **Host header** | `admin.example.com` | Multi-tenant routing |
| **HTTP method** | `POST` | Route writes to a different backend |
| **Query string** | `?version=2` | A/B testing or versioned APIs |
| **Source IP** | `10.0.0.0/8` | Internal-only access |
| **HTTP header** | `X-Internal: true` | Custom header–based routing |

### Setting Up Path-Based Routing via Console

1. Go to **EC2 → Load Balancers → select ALB → Listeners tab**.
2. Click on the HTTPS:443 listener → **Manage rules → Add rule**.
3. **Name** the rule (e.g., `route-api`).
4. Under **Conditions → Add condition**:
   - Select **Path** → enter `/api/*`.
5. Under **Actions → Add action**:
   - Select **Forward to target groups** → choose your API target group.
6. **Priority**: Lower number = evaluated first (e.g., priority 1 is checked before priority 100).
7. Click **Save**.

### Example Multi-Service Architecture

```
https://example.com/         ──► Frontend TG (React app, port 80)
https://example.com/api/*    ──► API TG (Node.js, port 3000)
https://example.com/auth/*   ──► Auth TG (Python, port 5000)
https://admin.example.com/*  ──► Admin TG (separate service, port 8080)
```

All served from a single ALB with a single ACM certificate (wildcard `*.example.com` + `example.com`).

### Weighted Target Groups (for Canary Deployments)

In the rule action, instead of forwarding to one target group, you can split traffic:

1. Action: **Forward to target groups**.
2. Add two target groups:
   - `production-tg` → Weight: 90
   - `canary-tg` → Weight: 10
3. 10% of traffic goes to the new version, 90% stays on the stable version.

---

## 15. Importing External Certificates into ACM

You may need to import certificates issued by an external CA (e.g., DigiCert, Sectigo, Let's Encrypt) into ACM to use them with ALB, NLB, or API Gateway.

### When to Import vs. Use ACM-Issued Certs

| Factor | ACM-Issued (Public) | Imported |
|---|---|---|
| Cost | Free | External CA cost applies |
| Renewal | Automatic | Manual (you must reimport) |
| Key control | AWS manages key | You manage key |
| Use case | Standard AWS services | Org-mandated CA, EV certs |

### Step-by-Step: Import a Certificate

1. Go to **Certificate Manager → Import a certificate**.
2. Paste the certificate bodies:
   - **Certificate body**: The PEM-encoded end-entity certificate (`.crt` or `.pem`).
   - **Certificate private key**: The PEM-encoded private key (unencrypted, no passphrase).
   - **Certificate chain**: Intermediate + root CA certificates (PEM, concatenated in order: intermediate first, root last).
3. Click **Next** → add tags → **Import**.

### Certificate Chain Order (Critical)

```
-----BEGIN CERTIFICATE-----
[Intermediate CA Certificate]
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
[Root CA Certificate]
-----END CERTIFICATE-----
```

Incorrect chain order causes browsers to show "untrusted" warnings even if the cert itself is valid.

### Renewal of Imported Certificates

ACM does **not** auto-renew imported certs. You must:

1. Obtain a renewed certificate from your CA.
2. Go to **Certificate Manager → select the cert → Reimport certificate**.
3. Paste the new cert body, key, and chain.
4. The certificate ARN stays the same — no changes needed to ALB/NLB/API GW configurations.

Set up an **EventBridge rule** to alert you when an imported cert is 45+ days from expiry:
- Event source: `aws.acm`
- Event type: `ACM Certificate Approaching Expiration`
- Target: SNS topic → email.

---

## 16. ACM Private CA — Deep Dive

ACM Private CA (also called AWS Private CA) is a managed private certificate authority service. Unlike public ACM certs (which are validated by a public CA and trusted by browsers), Private CA issues certificates for **internal use** within your organization.

### Use Cases

- Internal microservice communication (service-to-service TLS).
- mTLS client certificates.
- VPN endpoint certificates.
- IoT device authentication.
- Code signing (with appropriate cert profiles).

### CA Hierarchy

A proper PKI (Public Key Infrastructure) uses a hierarchy:

```
Root CA  (offline / air-gapped — most secure)
  └── Subordinate CA  (online — issues end-entity certs)
        ├── Server cert (example.internal.com)
        ├── Client cert (service-a)
        └── Client cert (service-b)
```

AWS Private CA supports both Root CAs and Subordinate CAs. Best practice: create a **Root CA** in AWS Private CA (or keep it external), then create a **Subordinate CA** for day-to-day issuance.

### Creating a Subordinate CA

1. **Certificate Manager → Private CA → Create a private CA**.
2. **CA type**: Subordinate CA.
3. **Subject**: Set meaningful organizational values.
4. **Key algorithm**: RSA 2048 or EC P256.
5. Click **Create CA** — the CA starts in **Pending** state.
6. Generate a **CSR** (Certificate Signing Request): **Actions → Get CSR**.
7. Sign the CSR with your Root CA:
   - If root is also in ACM Private CA: **Actions → Install subordinate CA certificate → ACM Private CA**.
   - If root is external: sign the CSR externally and import the signed certificate.
8. CA status changes to **Active**.

### Issuing Certificates via Console

1. **Certificate Manager → Request a certificate → Request a private certificate**.
2. Select the **Private CA**.
3. Enter the domain or service name.
4. **Certificate template**:
   - `EndEntityCertificate/V1` — standard server/client cert.
   - `CodeSigningCertificate/V1` — for code signing.
   - `RootCACertificate/V1` — for CA certs.
5. Set validity period.
6. Click **Request**.

### Pricing

- **Private CA**: ~$400/month per active CA (prorated hourly).
- **Certificates issued**: ~$0.75 per certificate (first 1,000/month).
- Free tier: None — costs apply immediately.

> Plan your CA hierarchy carefully — you pay per active CA per month regardless of whether you issue any certificates.

---

## 17. AWS WAF Integration with ALB TLS

AWS WAF (Web Application Firewall) sits in front of ALB and inspects decrypted HTTP traffic. Because TLS is terminated at the ALB before WAF sees the request, WAF works on plaintext — giving it full visibility into headers, body, and URI.

### Architecture

```
[Client] ──HTTPS──► [ALB (terminates TLS)] ──HTTP (decrypted)──► [WAF WebACL] ──► [Target Group]
```

WAF is applied as a **WebACL** associated with the ALB.

### Setting Up WAF on an ALB

#### Step 1: Create a WebACL

1. Go to **AWS WAF & Shield → Web ACLs → Create web ACL**.
2. **Name**: `my-alb-waf`.
3. **Resource type**: **Regional resources** (for ALB).
4. **Region**: Same as your ALB.
5. Click **Next**.

#### Step 2: Add Rules

Under **Add rules and rule groups**, you can add:

- **AWS Managed Rule Groups** (no extra config needed):
  - `AWSManagedRulesCommonRuleSet` — OWASP Top 10 protection.
  - `AWSManagedRulesSQLiRuleSet` — SQL injection protection.
  - `AWSManagedRulesKnownBadInputsRuleSet` — Log4j, Spring4Shell, etc.
- **Rate-based rules**: Block IPs that send more than N requests per 5 minutes.
- **Custom rules**: Match on specific headers, URIs, or IP sets.

#### Step 3: Associate WebACL with ALB

1. After creating the WebACL, go to **Associated AWS resources → Add association**.
2. Select **Application Load Balancer** → choose your ALB.
3. Save.

Alternatively, at the end of the WebACL wizard, add the ALB as an associated resource before saving.

#### Step 4: Enable WAF Logging (Optional but Recommended)

1. In the WebACL → **Logging and metrics** tab → **Enable logging**.
2. Destination: **CloudWatch Logs**, **S3**, or **Kinesis Data Firehose**.
3. Logs include: matched rule, action taken, source IP, URI, headers.

### WAF vs. Security Groups vs. NACLs

| Tool | Layer | Inspects | Best For |
|---|---|---|---|
| Security Group | L3/L4 | IP + Port | Allow/deny at instance/LB level |
| Network ACL | L3/L4 | IP + Port | Subnet-level stateless filtering |
| WAF | L7 | HTTP headers, body, URI | App-layer attacks (SQLi, XSS, bots) |

WAF complements, not replaces, security groups and NACLs.

---

## 18. Certificate Validation Methods

When you request an ACM public certificate, you must prove domain ownership.

### DNS Validation (Recommended)

ACM gives you a **CNAME record** to add to your DNS zone.

```
Name:  _abc123.example.com.
Type:  CNAME
Value: _xyz456.acm-validations.aws.
```

**Steps in AWS Console:**

1. After requesting the cert, click on it → status: **Pending validation**.
2. Expand the domain → click **Create records in Route 53** (if using Route 53 — one-click).
3. If using external DNS: manually copy and add the CNAME in your DNS provider.
4. Wait ~5–30 minutes for ACM to detect the record. Status → **Issued**.

**Advantage**: ACM uses this same CNAME for auto-renewal. Set it and forget it.

### Email Validation

ACM sends approval emails to:
- `admin@example.com`
- `administrator@example.com`
- `hostmaster@example.com`
- `webmaster@example.com`
- `postmaster@example.com`
- The domain WHOIS registrant contacts.

Click the approval link in the email. Certificate is issued within minutes.

**Disadvantage**: Requires manual action at renewal time too. Not recommended for production.

---

## 19. Security Best Practices

### TLS Policy Selection

Always use a modern security policy. Recommended:

| Policy | TLS Versions | Use Case |
|---|---|---|
| `ELBSecurityPolicy-TLS13-1-2-2021-06` | 1.2, 1.3 | **Default choice** — modern browsers |
| `ELBSecurityPolicy-TLS13-1-3-2021-06` | 1.3 only | Maximum security, may break older clients |
| `ELBSecurityPolicy-2016-08` | 1.0, 1.1, 1.2 | Legacy — avoid |

To update: **EC2 → Load Balancers → select ALB → Listeners tab → Edit listener → Security policy**.

### HTTPS Redirect

Always redirect HTTP to HTTPS. In ALB:

1. Add listener: **HTTP:80**.
2. Default action: **Redirect**.
3. Protocol: HTTPS, Port: 443, Status code: 301.

### HTTP Security Headers

Add response headers via ALB's response header modification (requires a listener rule with a fixed-response or via a WAF managed rule group):

- `Strict-Transport-Security: max-age=31536000; includeSubDomains`
- `X-Content-Type-Options: nosniff`
- `X-Frame-Options: DENY`
- `Content-Security-Policy: default-src 'self'`

You can also inject these headers at the application layer (e.g., Nginx `add_header` directives).

### Wildcard vs. Multi-Domain Certificates

- **Wildcard** (`*.example.com`): Covers all first-level subdomains. Cannot cover `example.com` itself (add it as a SAN alongside the wildcard).
- **Multi-domain (SAN)**: Add up to 10 domain names per cert. More explicit control.
- In ACM, you can add both `example.com` and `*.example.com` in one certificate.

### Principle of Least Privilege for ACM

Use IAM policies to control who can request, describe, delete, and associate ACM certificates:

```json
{
  "Effect": "Allow",
  "Action": [
    "acm:RequestCertificate",
    "acm:DescribeCertificate",
    "acm:ListCertificates"
  ],
  "Resource": "*"
}
```

Restrict `acm:DeleteCertificate` and `acm:ExportCertificate` (Private CA only) to privileged roles only.

### Certificate Monitoring

- Set up **EventBridge** rules for ACM certificate expiration (even though ACM auto-renews, imported certs do not).
- Rule event: `aws.acm` → `ACM Certificate Approaching Expiration`.
- Send to SNS topic → Email notification.

### Enable Access Logs on ALB

ALB access logs capture every request, including TLS details. Enable them:

1. **EC2 → Load Balancers → select ALB → Attributes tab → Edit**.
2. **Access logs**: Enable → specify S3 bucket and prefix.
3. Logs include: client IP, request time, request path, response code, SSL cipher, TLS protocol version, target response time.

Useful for security audits, debugging, and compliance.

---

## 20. Troubleshooting Common Issues

### Certificate Stuck in "Pending Validation"

- Verify the CNAME record was added correctly (check with `dig _abc123.example.com CNAME`).
- Ensure no proxy (e.g., Cloudflare orange-cloud) is interfering with the validation CNAME.
- DNS propagation can take up to 72 hours for external providers.

### "Certificate not in this region" Error

- ACM certs are regional. An ALB in `ap-south-1` cannot use a cert from `us-east-1`.
- Fix: Request a new cert in the correct region where your load balancer resides.

### 502 Bad Gateway from ALB

- ALB reached the target but got an invalid response.
- Check: Is the target group protocol correct? (HTTPS target group → backend must serve HTTPS.)
- Check: Is the backend certificate valid? (For HTTPS target groups, ALB validates the backend cert by default.)
- Check: Security group on EC2 allows the ALB's security group on the target port.

### 504 Gateway Timeout

- ALB cannot reach the backend at all.
- Check: EC2 security group allows inbound from ALB SG.
- Check: Target is registered and healthy in target group.
- Check: Application is actually running and listening on the target port.

### NLB Source IP Not Preserved

- NLB in TLS passthrough (TCP listener) preserves client IP natively.
- If using Proxy Protocol v2, configure both the NLB attribute AND your backend server to parse it.
- NLB → Attributes → **Proxy protocol v2**: Enable. Then configure Nginx: `proxy_protocol on;`.

### mTLS: Client Certificate Rejected

- Check that the client certificate was issued by a CA in the ALB Trust Store.
- Verify the cert has not expired: `openssl x509 -in client.crt -noout -dates`.
- Confirm the Trust Store contains the full chain (intermediate + root).
- Check ALB access logs for the rejection reason: look for `tls_verify_status` field.

### ACM Certificate Not Renewing

- DNS validation: Ensure the ACM CNAME record is still present in DNS. If removed, renewal fails.
- Email validation: Renewal emails are sent 45 days before expiry; check spam folder.
- Check certificate status in the console — if `Renewal Eligibility: Ineligible`, the cert may have been imported or the domain is no longer associated with an active AWS resource.

### Imported Certificate Chain Error

- Test the chain with: `openssl verify -CAfile chain.pem certificate.crt`
- Ensure intermediates are in order: leaf → intermediate(s) → root.
- Browsers may cache the old cert; test in incognito or with: `openssl s_client -connect example.com:443 -showcerts`

---

## 21. ACM Limits & Pricing Reference

### ACM Public Certificate Limits (per region, per account)

| Resource | Default Limit | Can Increase? |
|---|---|---|
| Public certificates | 2,500 | Yes (AWS Support) |
| Domain names per certificate | 10 | No |
| Certificates per ALB listener | 25 (+ 1 default) | No |
| Certificates per NLB listener | 1 (default only) | No |
| Pending validations per domain | 10 | No |

### ACM Private CA Limits

| Resource | Default Limit |
|---|---|
| Private CAs per region | 200 |
| Certificates issued per CA per month | 50,000 |
| CA validity period | Up to 30 years |

### Pricing Summary

| Item | Price |
|---|---|
| ACM Public Certificates | **Free** |
| Imported Certificates | **Free** (ACM hosting cost) |
| ACM Private CA — CA operation | ~$400/month per active CA |
| ACM Private CA — Certificates issued | ~$0.75 each (first 1,000/month) |
| ALB — per hour | ~$0.008/hour + LCU charges |
| NLB — per hour | ~$0.006/hour + NLCU charges |
| WAF WebACL | ~$5/month + $1/rule/month + $0.60/million requests |

> Prices vary by region. Always check the [AWS Pricing page](https://aws.amazon.com/certificate-manager/pricing/) for current rates.

### Certificate Lifecycle Summary

```
Request → Pending Validation → Issued → [In Use] → Approaching Expiry → Auto-Renewed
                                                                    ↑
                                               (DNS record must still be present)
```

For imported certs, the lifecycle ends at "Approaching Expiry" — you must reimport manually.

---

## 22. Quick Reference Comparison Table

| Scenario | Load Balancer | Listener Protocol | TLS Terminated By | ACM Cert Used | Backend Protocol |
|---|---|---|---|---|---|
| 1. ALB Edge Termination | ALB | HTTPS | ALB | Yes | HTTP |
| 2. API Gateway Custom Domain | API GW | HTTPS | API Gateway | Yes | HTTP/Lambda |
| 3. NLB TCP Passthrough | NLB | TCP | Backend EC2 | No (self-managed) | HTTPS |
| 4. NLB TLS Termination | NLB | TLS | NLB | Yes | TCP (HTTP) |
| 5. End-to-End Re-encryption | ALB | HTTPS | ALB + Backend | ACM (ALB side) | HTTPS |
| 6. ECS Fargate + ALB | ALB | HTTPS | ALB | Yes | HTTP (container) |
| 7. mTLS with Private CA | ALB | HTTPS (mTLS) | ALB | Yes + Private CA | HTTP |
| 8. Elastic Beanstalk | ALB (EB-managed) | HTTPS | ALB | Yes | HTTP |
| 9. App Runner | Built-in | HTTPS | App Runner | Auto-provisioned | HTTP (internal) |

---

## Summary

AWS ACM combined with ALB, NLB, and API Gateway covers virtually every TLS termination pattern needed in modern cloud architectures:

- **Simplest setup**: ALB + ACM (Scenario 1) handles 80% of use cases.
- **APIs**: API Gateway custom domains (Scenario 2).
- **Compliance/mTLS**: NLB passthrough (Scenario 3) or re-encryption (Scenario 5).
- **High throughput / static IP**: NLB TLS termination (Scenario 4).
- **Containers**: ECS/Fargate always terminates at the ALB (Scenario 6).
- **Zero-trust / B2B**: mTLS with ACM Private CA (Scenario 7).
- **Managed platforms**: Beanstalk and App Runner handle the heavy lifting (Scenarios 8–9).
- **Multi-domain on one listener**: SNI with multiple ACM certs on ALB.
- **App-layer protection**: WAF WebACL associated with ALB.

ACM's automatic renewal, zero-cost public certificates, and deep integration with AWS services make it the standard choice for TLS management on AWS.
