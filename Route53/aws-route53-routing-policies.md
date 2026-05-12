# AWS Route 53 — Complete Guide to Routing Policies

> A thorough, hands-on reference covering Hosted Zones, NS records, A records with Alias, and every routing policy — with setup steps, real-world examples, pros, cons, and commentary on *why* each step matters.

---

## Table of Contents

1. [What is Route 53?](#1-what-is-route-53)
2. [Hosted Zones](#2-hosted-zones)
3. [NS Records & Changing Name Servers](#3-ns-records--changing-name-servers)
4. [Record Types Overview](#4-record-types-overview)
5. [A Record with Alias](#5-a-record-with-alias)
6. [Routing Policies](#6-routing-policies)
   - [Simple Routing](#61-simple-routing)
   - [Weighted Routing](#62-weighted-routing)
   - [Latency-Based Routing](#63-latency-based-routing)
   - [Failover Routing](#64-failover-routing)
   - [Geolocation Routing](#65-geolocation-routing)
   - [Geoproximity Routing](#66-geoproximity-routing)
   - [Multivalue Answer Routing](#67-multivalue-answer-routing)
   - [IP-Based Routing](#68-ip-based-routing)
7. [Health Checks](#7-health-checks)
8. [Comparison Table](#8-comparison-table)
9. [Common Pitfalls](#9-common-pitfalls)
10. [Real-World Architecture Example](#10-real-world-architecture-example)

---

## 1. What is Route 53?

Amazon Route 53 is AWS's highly available, scalable DNS (Domain Name System) web service. It does three main things:

| Function | Description |
|---|---|
| **Domain Registration** | Buy and manage domain names directly through AWS |
| **DNS Resolution** | Translate human-friendly names (e.g., `www.example.com`) into IP addresses |
| **Health Checking** | Monitor endpoints and route traffic away from unhealthy ones |

The name "Route 53" is a nod to port **53**, the standard UDP/TCP port used by DNS.

---

## 2. Hosted Zones

### What is a Hosted Zone?

A **Hosted Zone** is a container for DNS records for a specific domain. Think of it as the DNS "folder" for your domain. Every domain you manage in Route 53 lives inside a Hosted Zone.

There are two types:

| Type | Description | Use Case |
|---|---|---|
| **Public Hosted Zone** | Resolves DNS queries from the public internet | Public-facing websites, APIs |
| **Private Hosted Zone** | Resolves DNS queries only within one or more VPCs | Internal microservices, databases, intranet |

### Setting Up a Public Hosted Zone

**Step 1 — Create the Hosted Zone**

```
AWS Console → Route 53 → Hosted Zones → Create Hosted Zone
  Domain name:  example.com
  Type:         Public Hosted Zone
  Comment:      My production domain (optional)
```

**Why?** This tells Route 53 "I want to manage DNS for `example.com`." Without a Hosted Zone, you cannot create any DNS records for that domain.

**Step 2 — Inspect auto-created records**

When a Hosted Zone is created, AWS automatically adds two record sets:

- **NS record** — lists the four Route 53 name servers assigned to your zone
- **SOA record** — Start of Authority; contains metadata like the primary DNS server and TTL defaults

**Why?** These records are mandatory for DNS to function. The NS record is what the rest of the internet uses to discover *where* to send DNS queries for your domain.

**Example — NS record (auto-generated):**

```
Name:   example.com
Type:   NS
TTL:    172800 (48 hours)
Values:
  ns-123.awsdns-45.com
  ns-678.awsdns-90.net
  ns-111.awsdns-22.org
  ns-999.awsdns-01.co.uk
```

**Step 3 — Set up a Private Hosted Zone (optional)**

```
AWS Console → Route 53 → Hosted Zones → Create Hosted Zone
  Domain name:  internal.example.com
  Type:         Private Hosted Zone
  VPC:          vpc-0abc123def456789  (your VPC)
  Region:       ap-south-1
```

**Why?** Internal services (RDS, ElastiCache, internal APIs) should not be resolvable from the public internet. A private zone ensures only resources inside your VPC can resolve these names.

---

## 3. NS Records & Changing Name Servers

### What are NS Records?

**Name Server (NS) records** tell the world *which DNS servers are authoritative* for your domain. When someone types `example.com`, the global DNS system checks your domain registrar for NS records, then asks *those* servers for the actual IP.

### Why You Need to Change NS Records

If you registered your domain at a third-party registrar (GoDaddy, Namecheap, Google Domains, etc.) and created a Hosted Zone in Route 53, your domain still points to the registrar's name servers — **Route 53 won't be consulted until you update the NS records at your registrar.**

### Step-by-Step: Delegating to Route 53

**Step 1 — Copy your Route 53 NS values**

```
Route 53 → Hosted Zones → example.com → NS record
```

Copy all four name server addresses. They look like:
```
ns-1234.awsdns-28.org
ns-567.awsdns-09.net
ns-89.awsdns-11.com
ns-910.awsdns-45.co.uk
```

**Step 2 — Log in to your domain registrar**

Navigate to DNS settings / Name Server settings for your domain.

**Step 3 — Replace the existing NS values**

Remove the registrar's default name servers and paste the four Route 53 NS values.

**Why?** Without this step, DNS queries for your domain go to the registrar's servers, which know nothing about your Route 53 records. The site will appear as if it doesn't exist or show the registrar's parked page.

**Step 4 — Wait for propagation**

DNS propagation can take **5 minutes to 48 hours** depending on the TTL of the old NS records and how aggressively DNS resolvers cache.

**Pro tip:** Use `dig NS example.com` or `nslookup -type=NS example.com` to verify propagation.

```bash
# Verify NS records are pointing to Route 53
dig NS example.com +short

# Expected output:
ns-1234.awsdns-28.org.
ns-567.awsdns-09.net.
ns-89.awsdns-11.com.
ns-910.awsdns-45.co.uk.
```

**Pros of using Route 53 as your DNS:**
- 100% SLA uptime for DNS resolution
- Native integration with AWS services (ALB, CloudFront, S3, etc.)
- Health-check-aware routing
- Low latency via Anycast network (Route 53 has 100+ PoPs globally)

**Cons:**
- Small cost per hosted zone ($0.50/month) and per million queries
- Propagation delay when changing name servers
- Vendor lock-in for DNS management

---

## 4. Record Types Overview

| Record Type | Purpose | Example |
|---|---|---|
| **A** | Maps hostname → IPv4 address | `app.example.com → 203.0.113.10` |
| **AAAA** | Maps hostname → IPv6 address | `app.example.com → 2001:db8::1` |
| **CNAME** | Maps hostname → another hostname | `www.example.com → app.example.com` |
| **Alias** | AWS-specific; maps hostname → AWS resource DNS name | `example.com → my-alb.us-east-1.elb.amazonaws.com` |
| **MX** | Mail server routing | `example.com → mail.example.com` |
| **TXT** | Text records (SPF, DKIM, domain verification) | `example.com → "v=spf1 include:..."` |
| **NS** | Name server delegation | `example.com → ns-1234.awsdns-28.org` |
| **PTR** | Reverse DNS lookup | `10.113.0.203.in-addr.arpa → app.example.com` |
| **SRV** | Service location records | Used for SIP, XMPP, etc. |
| **CAA** | Certificate Authority Authorization | Restricts which CAs can issue SSL certs |

---

## 5. A Record with Alias

### Regular A Record vs. Alias A Record

This is one of Route 53's most important — and most misunderstood — distinctions.

| | Regular A Record | Alias A Record |
|---|---|---|
| **Points to** | IPv4 address (e.g., `203.0.113.10`) | AWS resource DNS name (e.g., ALB, CloudFront, S3) |
| **Works at zone apex** | ✅ Yes | ✅ Yes |
| **CNAME at zone apex** | ❌ Not allowed in DNS spec | N/A |
| **Charges for queries** | Yes | **No** — Alias queries are free |
| **TTL** | You set it | Route 53 manages it automatically |
| **Health check support** | Via separate health checks | Can inherit target's health check |
| **Target changes IP** | You must update manually | Route 53 follows automatically |

### Why Alias Records Exist

The DNS specification forbids CNAME records at the **zone apex** (the root domain, e.g., `example.com` without any subdomain). This is a problem because AWS resources like ALBs and CloudFront distributions give you a DNS name, not a static IP.

AWS invented **Alias records** to solve this — they work like CNAME under the hood but are fully valid at the zone apex.

### Setting Up an A Record (Regular)

**Use case:** You have an EC2 instance with Elastic IP `52.14.200.10` and want `app.example.com` to point to it.

```
Route 53 → Hosted Zone → Create Record
  Record name:  app
  Record type:  A
  Value:        52.14.200.10
  TTL:          300 (5 minutes)
  Routing:      Simple
```

**Why set TTL to 300?** Lower TTL means DNS resolvers refresh more often — useful during deployments when you want quick failover. High TTL (like 86400) reduces DNS query load but slows down propagation of changes.

```bash
# Verify
dig A app.example.com +short
# Output: 52.14.200.10
```

### Setting Up an A Record with Alias

**Use case:** Point the root domain `example.com` to an Application Load Balancer.

```
Route 53 → Hosted Zone → Create Record
  Record name:  (leave blank — this is the zone apex)
  Record type:  A
  Alias:        Yes (toggle on)
  Route traffic to:
    → Alias to Application and Classic Load Balancer
    → Region: us-east-1
    → Load balancer: my-app-alb-1234567890.us-east-1.elb.amazonaws.com
  Routing policy: Simple
```

**Why use Alias here?**
1. You **cannot** use a CNAME at the zone apex (`example.com`) — DNS spec forbids it.
2. ALBs don't have static IPs — their IPs can change as AWS scales them. Alias records automatically follow these IP changes.
3. Alias queries to AWS resources are **free** — you pay $0 per DNS query instead of the usual $0.40 per million queries.

**Example — Alias to S3 static website:**

```
Record name:  www
Record type:  A
Alias:        Yes
Route traffic to:
  → Alias to S3 website endpoint
  → Region: ap-south-1
  → Bucket: www.example.com  (bucket must be named exactly like the record)
```

**Example — Alias to CloudFront:**

```
Record name:  (apex)
Record type:  A
Alias:        Yes
Route traffic to:
  → Alias to CloudFront distribution
  → d1234abcd.cloudfront.net
```

**Pros of Alias:**
- Free DNS queries for AWS-target aliases
- Automatically tracks IP changes in ALBs, CloudFront, etc.
- Works at zone apex (root domain)
- Can participate in health-check-based routing

**Cons of Alias:**
- Only works with specific AWS services (ALB, NLB, CloudFront, Elastic Beanstalk, API Gateway, S3, Global Accelerator, VPC endpoints)
- Cannot point to on-premises or non-AWS endpoints
- TTL is not configurable — Route 53 controls it

---

## 6. Routing Policies

Route 53 supports **8 routing policies**. Each policy determines *how* Route 53 responds to DNS queries for a given record name.

---

### 6.1 Simple Routing

**What it does:** Returns one or more IP addresses for a single resource. No health checks. No conditional logic.

**When to use:**
- Development/staging environments
- Single-server setups
- When you want pure DNS with zero complexity

**Setup:**

```
Route 53 → Create Record
  Name:     www
  Type:     A
  Value:    203.0.113.10
  TTL:      300
  Routing:  Simple
```

**Multiple values in Simple Routing:**

You can add multiple IPs in a single Simple record. Route 53 returns all of them, and the client picks one randomly (client-side load balancing):

```
Name:     www
Type:     A
Values:
  203.0.113.10
  203.0.113.20
  203.0.113.30
TTL:      60
Routing:  Simple
```

**Why?** This gives you primitive load distribution without any AWS-specific features. The browser or OS randomly picks one IP from the list.

**Important:** Simple routing does **not** support health checks on individual values. If one IP is down, Route 53 still returns it.

**Example — Verify with dig:**

```bash
dig A www.example.com
# Answer section may show multiple A records:
# www.example.com.  60  IN  A  203.0.113.10
# www.example.com.  60  IN  A  203.0.113.20
# www.example.com.  60  IN  A  203.0.113.30
```

**Pros:**
- Simplest to set up
- No cost beyond standard query fees
- Predictable behavior

**Cons:**
- No health checks
- No intelligent traffic distribution
- If server is down, clients still get the dead IP
- Not suitable for production multi-server setups

---

### 6.2 Weighted Routing

**What it does:** Distributes traffic across multiple resources based on assigned weights (percentages). Useful for blue/green deployments, A/B testing, and gradual traffic shifting.

**Formula:** `traffic to record = (record weight) / (sum of all weights)`

**When to use:**
- Canary deployments (send 5% to new version, 95% to old)
- A/B testing
- Gradually migrating from one server/region to another
- Load testing a new deployment with a small traffic slice

**Setup — Blue/Green Deployment Example:**

You have two versions of your app: `blue` (current) on `203.0.113.10` and `green` (new) on `203.0.113.20`. Start by sending 90% to blue, 10% to green.

**Record 1 — Blue (production):**

```
Route 53 → Create Record
  Name:           www
  Type:           A
  Value:          203.0.113.10
  TTL:            60
  Routing policy: Weighted
  Weight:         90
  Record ID:      blue-v1        ← must be unique per weighted set
  Health check:   hc-blue        ← associate a health check
```

**Record 2 — Green (new version):**

```
Route 53 → Create Record
  Name:           www
  Type:           A
  Value:          203.0.113.20
  TTL:            60
  Routing policy: Weighted
  Weight:         10
  Record ID:      green-v2
  Health check:   hc-green
```

**Why set TTL to 60?** During a deployment, you want DNS caches to expire quickly so the new weights take effect fast.

**Why use a Record ID?** Weighted records sharing the same name and type must each have a unique Record ID. Route 53 uses this to distinguish them internally.

**Gradually shift traffic:**

```
Day 1:  Blue=90, Green=10
Day 2:  Blue=70, Green=30
Day 3:  Blue=50, Green=50
Day 4:  Blue=10, Green=90
Day 5:  Blue=0,  Green=100  → decommission blue
```

**Special case — weight of 0:**

Setting a record's weight to `0` effectively removes it from rotation without deleting the record. Route 53 stops returning it. Useful for emergency removal.

**Example CLI:**

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "www.example.com",
        "Type": "A",
        "SetIdentifier": "green-v2",
        "Weight": 30,
        "TTL": 60,
        "ResourceRecords": [{"Value": "203.0.113.20"}]
      }
    }]
  }'
```

**Pros:**
- Fine-grained traffic control
- Supports health checks per record
- Great for zero-downtime deployments
- Can mimic blue/green or canary workflows without complex infrastructure

**Cons:**
- DNS caching means percentages are approximate, not exact
- Doesn't account for actual server load (a weak server gets the same weight as a strong one unless you adjust manually)
- Requires careful weight management to avoid accidental traffic loss

---

### 6.3 Latency-Based Routing

**What it does:** Routes users to the AWS region that provides the **lowest network latency** for them. Route 53 measures latency between the resolver's location and each AWS region and picks the best.

**When to use:**
- Multi-region deployments
- Global applications where speed matters
- When your users are spread across continents

**Setup — Multi-region App (us-east-1, eu-west-1, ap-south-1):**

**Record 1 — US East:**

```
Name:           api
Type:           A (or Alias to ALB)
Value:          54.10.20.30      ← US ALB IP or Alias
Routing policy: Latency
Region:         us-east-1
Record ID:      api-us-east
Health check:   hc-us-east
```

**Record 2 — Europe:**

```
Name:           api
Type:           A
Value:          52.50.60.70      ← EU ALB IP
Routing policy: Latency
Region:         eu-west-1
Record ID:      api-eu-west
Health check:   hc-eu-west
```

**Record 3 — Asia Pacific (Mumbai):**

```
Name:           api
Type:           A
Value:          13.230.40.50     ← APAC ALB IP
Routing policy: Latency
Region:         ap-south-1
Record ID:      api-ap-south
Health check:   hc-ap-south
```

**Why include health checks?** If `us-east-1` goes down, Route 53 should stop routing US users there and fall back to the next-lowest-latency region. Without health checks, users get sent to a dead endpoint.

**How latency is measured:**

Route 53 maintains a database of historical latency measurements between all AWS regions and all CIDR blocks worldwide. It doesn't ping your server in real time — it uses precomputed latency data. This means latency routing reflects *network path quality*, not your server's response time.

**Verify routing from different locations:**

```bash
# From a server in Mumbai — should resolve to ap-south-1 IP
dig api.example.com @8.8.8.8

# From a server in London — should resolve to eu-west-1 IP
dig api.example.com @8.8.8.8
```

**Pros:**
- Significantly improves user experience for global audiences
- Automatic — no user needs to know which region to use
- Combines with health checks for resilience
- Works with Alias records pointing to region-specific ALBs

**Cons:**
- Doesn't measure your actual application latency — only network path latency
- Users may not always go to the region you expect (latency data is approximate)
- More complex to set up and debug than simple routing
- All regions must have identical application deployments for this to work correctly

---

### 6.4 Failover Routing

**What it does:** Routes traffic to a **primary** resource when it's healthy, and automatically switches to a **secondary** (standby) resource when the primary fails a health check.

**When to use:**
- Active-passive disaster recovery
- Ensuring a backup site is reachable if your primary goes down
- Any setup where you need guaranteed failover

**Architecture:**

```
           ┌─────────────────────────────────┐
           │         Route 53                │
           │   Failover Routing Policy        │
           └───────────┬─────────────────────┘
                       │
         ┌─────────────┴─────────────┐
         ▼                           ▼
  [Primary - us-east-1]       [Secondary - eu-west-1]
   Healthy → gets traffic       Standby → gets traffic
                                 only if primary fails
```

**Setup:**

**Step 1 — Create a health check for the primary:**

```
Route 53 → Health Checks → Create Health Check
  Name:       hc-primary
  Monitor:    Endpoint
  Protocol:   HTTPS
  IP/Domain:  203.0.113.10
  Path:       /health
  Port:       443
  Interval:   10 seconds  ← fast detection
  Threshold:  2 failures  ← fail after 2 consecutive failures
```

**Why `/health`?** Always create a dedicated health endpoint in your app that checks internal dependencies (DB connectivity, cache, etc.) and returns HTTP 200 only when truly healthy. Don't health check `/` — it may return 200 even when the app is broken.

**Step 2 — Primary record:**

```
Name:           www
Type:           A
Value:          203.0.113.10
Routing policy: Failover
Failover type:  Primary
Record ID:      www-primary
Health check:   hc-primary
```

**Step 3 — Secondary record:**

```
Name:           www
Type:           A
Value:          203.0.113.50    ← DR/backup server
Routing policy: Failover
Failover type:  Secondary
Record ID:      www-secondary
Health check:   hc-secondary   ← optional but recommended
```

**Why health check the secondary too?** If the secondary is also down when failover occurs, Route 53 will still return it (there's no tertiary). Monitoring both lets you react faster.

**Failover timeline:**

```
T+0:   Primary fails health check (1st failure)
T+10s: Primary fails health check (2nd failure) → marked unhealthy
T+20s: Route 53 stops returning primary IP
T+20s: Secondary IP returned for all new DNS queries
T+80s: Most cached DNS responses expire (TTL 60s) → all traffic on secondary
```

**Pros:**
- Simple active-passive setup
- Built into Route 53 — no external tools needed
- Fast detection with 10-second health check intervals
- Ideal for DR scenarios

**Cons:**
- Secondary is idle (wasting resources) in active-passive
- DNS TTL means failover isn't instant — there's a propagation delay
- Only two tiers (primary + secondary) — no tertiary
- For active-active, use Weighted or Latency routing instead

---

### 6.5 Geolocation Routing

**What it does:** Routes traffic based on the **geographic location of the DNS resolver** (which roughly correlates to the user's location). You define rules like "users from India → India servers, users from US → US servers."

**When to use:**
- Serving country/region-specific content (language, regulations, pricing)
- GDPR compliance — route EU users to EU data centers
- Content licensing restrictions (e.g., streaming rights by country)
- Localizing user experience

**Hierarchy of geolocation matching:**
1. Country (most specific)
2. Continent
3. Default (catch-all for unmatched locations)

**Setup — Global App with Regional Routing:**

**Record 1 — India:**

```
Name:           www
Type:           A (Alias to ALB in ap-south-1)
Routing policy: Geolocation
Location:       Country → India
Record ID:      www-india
Health check:   hc-india
```

**Record 2 — European Union (multiple countries):**

You must add one record per country, or use a continent. Example: route all of Europe to EU servers:

```
Name:           www
Type:           A
Routing policy: Geolocation
Location:       Continent → Europe
Record ID:      www-europe
Health check:   hc-europe
```

**Record 3 — United States:**

```
Name:           www
Type:           A
Routing policy: Geolocation
Location:       Country → United States
Record ID:      www-us
Health check:   hc-us
```

**Record 4 — Default (catch-all):**

```
Name:           www
Type:           A
Routing policy: Geolocation
Location:       Default
Record ID:      www-default
Health check:   hc-default
```

**Why create a Default record?** Without it, users from unmapped locations (e.g., Antarctica) get a `NODATA` response and cannot access your site at all.

**Important:** Geolocation is based on the **resolver's IP**, not the user's IP. Users on VPNs may be routed to the wrong region. This is a fundamental DNS limitation.

**Example use case — GDPR:**

```
EU users       → eu-west-1 (data stays in EU)
All others     → us-east-1
```

```bash
# Simulate query from a specific location using a regional DNS resolver
dig www.example.com @8.8.8.8       # Google DNS — likely routes to default/US
dig www.example.com @49.205.0.1    # BSNL India resolver — likely routes to India
```

**Pros:**
- Precise geographic control
- Enforce data residency requirements (GDPR, HIPAA)
- Serve localized content without application-level logic
- Country-level granularity

**Cons:**
- VPN users get routed based on VPN exit location
- Geolocation accuracy depends on IANA/MaxMind databases — not 100% precise
- More records to manage (one per location)
- Does not optimize for latency — a user in Russia might be closer to a US server than a EU server, but geolocation ignores that

---

### 6.6 Geoproximity Routing

**What it does:** Routes traffic based on the **geographic distance** between users and your resources, with an optional **bias** that lets you expand or shrink the catchment area of a resource.

**When to use:**
- You want latency-like routing but need to manually adjust traffic boundaries
- Shifting traffic from one region to another gradually based on geographic area
- Fine-tuning traffic distribution between regions that are close together

**Difference from Geolocation:**

| | Geolocation | Geoproximity |
|---|---|---|
| **Routing basis** | User's country/continent | Distance from resource |
| **Bias adjustment** | ❌ No | ✅ Yes |
| **Use case** | Legal/compliance | Optimizing geographic coverage |

**Bias values:**
- **Positive bias (+1 to +99):** Expands the region's catchment area — attracts more traffic from farther away
- **Negative bias (-1 to -99):** Shrinks the region — pushes traffic to neighboring regions
- **0:** Default, based purely on distance

**Prerequisite:** Geoproximity routing requires **Route 53 Traffic Flow** (a visual traffic policy editor). It cannot be set up with standard record creation.

**Setup via Traffic Flow:**

```
Route 53 → Traffic Policies → Create Traffic Policy
  Name:     geoproximity-policy
  Version:  1

In the visual editor:
  Add rule: Geoproximity
    Endpoint 1: us-east-1 ALB  | Bias: +20  (attract East Coast + more)
    Endpoint 2: eu-west-1 ALB  | Bias: 0
    Endpoint 3: ap-south-1 ALB | Bias: +10

Apply policy to: www.example.com
```

**Bias example:**

Without bias, a user in Brazil might go to `us-east-1`. Add `+20` bias to `sa-east-1` (São Paulo) and those users shift to the South America endpoint instead.

**Pros:**
- Most flexible geographic routing
- Bias allows soft traffic migration between regions
- Good for gradual regional expansion
- Works with both AWS resources and on-premises endpoints

**Cons:**
- Requires Traffic Flow (additional cost: $50/month per policy record)
- More complex to configure and visualize
- Bias tuning requires experimentation
- Overkill for most standard setups

---

### 6.7 Multivalue Answer Routing

**What it does:** Returns up to **8 healthy IP addresses** in response to a DNS query. Unlike Simple routing with multiple values, Multivalue Answer routing checks health for **each individual record** and only returns healthy ones.

**When to use:**
- Client-side load balancing across multiple servers
- Poor man's load balancer (not a replacement for ALB/NLB)
- When you have 2–8 servers and want basic redundancy without an ELB

**Key difference from Simple Routing:**

| | Simple (multiple values) | Multivalue Answer |
|---|---|---|
| **Health checks** | ❌ Not per-record | ✅ Per record |
| **Max values** | Unlimited | 8 |
| **Unhealthy removal** | ❌ Still returned | ✅ Excluded from response |
| **Record ID required** | ❌ No | ✅ Yes |

**Setup — 3 Web Servers:**

**Server 1:**

```
Name:           www
Type:           A
Value:          203.0.113.10
TTL:            60
Routing policy: Multivalue Answer
Record ID:      www-server-1
Health check:   hc-server-1
```

**Server 2:**

```
Name:           www
Type:           A
Value:          203.0.113.20
TTL:            60
Routing policy: Multivalue Answer
Record ID:      www-server-2
Health check:   hc-server-2
```

**Server 3:**

```
Name:           www
Type:           A
Value:          203.0.113.30
TTL:            60
Routing policy: Multivalue Answer
Record ID:      www-server-3
Health check:   hc-server-3
```

**How it works:**

```
DNS query for www.example.com →
  Route 53 checks health of all 3 servers →
  Server 2 is unhealthy →
  Route 53 returns: [203.0.113.10, 203.0.113.30]  (only healthy ones)
  Client randomly picks one
```

```bash
# Run multiple times — you'll get different orderings (random)
dig A www.example.com +short
# 203.0.113.10
# 203.0.113.30

dig A www.example.com +short
# 203.0.113.30
# 203.0.113.10
```

**Important:** Multivalue Answer is **not a replacement for a load balancer**. It's DNS-level, so it doesn't handle SSL termination, sticky sessions, or connection draining. Use it as a supplement, not a substitute.

**Pros:**
- Simple redundancy without an ELB
- Health checks filter out dead servers automatically
- Up to 8 healthy IPs returned
- Costs less than running a load balancer

**Cons:**
- DNS caching means clients may hold onto an IP after it becomes unhealthy (TTL delay)
- No true load balancing — client picks randomly, not by server load
- Max 8 records
- Not suitable for stateful sessions without sticky routing

---

### 6.8 IP-Based Routing

**What it does:** Routes traffic based on the **IP address of the DNS resolver** (or optionally the client's IP when EDNS Client Subnet is supported). You define CIDR blocks and map them to specific endpoints.

**When to use:**
- You have precise knowledge of where your users' IP ranges come from
- Routing corporate offices (fixed IP ranges) to specific endpoints
- Migrating customers from on-prem IP blocks to new infrastructure
- ISP-specific routing (route Comcast users to this server, Jio users to that one)

**Prerequisite:** You create **CIDR collections** first, then reference them in routing rules.

**Setup:**

**Step 1 — Create a CIDR Collection:**

```
Route 53 → CIDR Collections → Create Collection
  Name: corporate-offices

Add CIDR blocks:
  203.0.113.0/24   ← Mumbai office IP range
  198.51.100.0/24  ← Singapore office IP range
```

**Step 2 — Create IP-Based Records:**

**Record for corporate offices:**

```
Name:           app
Type:           A
Value:          10.0.1.50        ← Internal/private app server
Routing policy: IP-based
CIDR collection: corporate-offices
Record ID:      app-corp
```

**Record for everyone else (default):**

```
Name:           app
Type:           A
Value:          203.0.113.100    ← Public app server
Routing policy: IP-based
CIDR collection: (none — default)
Record ID:      app-public
```

**Why?** Corporate users on known IP ranges get routed to an internal or premium server. Everyone else hits the public endpoint.

**Example — ISP routing:**

```
Jio India CIDR block → ap-south-1 edge server (optimized for Jio peering)
Airtel India CIDR     → different ap-south-1 edge (optimized for Airtel peering)
Default               → standard server
```

**Pros:**
- Most precise routing control available
- Great for corporate/enterprise traffic management
- Bypass geographic ambiguity of geolocation routing
- Can be used for security (route known-malicious IPs to a honeypot)

**Cons:**
- Requires knowing your users' IP ranges upfront
- CIDR collections must be maintained as IP ranges change
- Complex to set up and maintain
- Doesn't work well for consumer traffic (dynamic IPs)
- Newest policy — less community documentation

---

## 7. Health Checks

Health checks are the backbone of any resilient Route 53 setup. They determine whether a record is included in DNS responses.

### Types of Health Checks

| Type | What it monitors | Use case |
|---|---|---|
| **HTTP/HTTPS endpoint** | Checks if a URL returns 2xx/3xx | Web servers, APIs |
| **TCP endpoint** | Checks if a port accepts connections | Databases, non-HTTP services |
| **CloudWatch Alarm** | Monitors a CloudWatch metric | Custom application metrics |
| **Calculated** | Aggregates multiple health checks (AND/OR logic) | Complex dependencies |

### Setting Up a Health Check

```
Route 53 → Health Checks → Create Health Check
  Name:                hc-production-api
  Monitor:             Endpoint
  Specify endpoint by: IP address
  Protocol:            HTTPS
  IP address:          203.0.113.10
  Host name:           api.example.com    ← sent as HTTP Host header
  Port:                443
  Path:                /health
  Advanced:
    Request interval:  10 seconds  (standard: 30s, fast: 10s)
    Failure threshold: 3           (mark unhealthy after 3 consecutive failures)
    String matching:   "healthy"   (optional: check response body)
    Enable SNI:        Yes         (required for HTTPS with virtual hosting)
```

**Why `/health` and not `/`?** A dedicated health endpoint can check internal system state:

```python
# Example Flask health endpoint
@app.route('/health')
def health():
    try:
        db.session.execute('SELECT 1')     # Check DB connectivity
        redis_client.ping()                # Check cache
        return jsonify({"status": "healthy"}), 200
    except Exception as e:
        return jsonify({"status": "unhealthy", "error": str(e)}), 503
```

### Calculated Health Checks

Combine multiple health checks with boolean logic:

```
hc-web-server    HTTPS check on port 443
hc-database      TCP check on port 5432
hc-cache         TCP check on port 6379

hc-overall = hc-web-server AND hc-database AND hc-cache
             (healthy only if all three are healthy)
```

```
Route 53 → Create Health Check
  Type:   Calculated
  Logic:  HEALTHY when 3 of 3 health checks are healthy
  Checks: hc-web-server, hc-database, hc-cache
```

**Route 53 Health Checker IPs:**

Route 53 checks your endpoints from approximately 15 global health checker locations. Ensure your server/security groups allow traffic from:
```
54.183.255.128/26
54.228.16.0/26
54.232.40.64/26
176.34.159.192/26
54.241.32.64/26
... (full list at https://ip-ranges.amazonaws.com/ip-ranges.json)
```

**Cost:** $0.50/month per health check endpoint (first 50 free with Route 53 registered domains).

---

## 8. Comparison Table

| Policy | Basis | Health Checks | Best For | Traffic Flow Required |
|---|---|---|---|---|
| **Simple** | None — single record | ❌ | Dev/single server | ❌ |
| **Weighted** | Numeric weights | ✅ Per record | A/B testing, canary deploy | ❌ |
| **Latency** | AWS region latency | ✅ Per record | Global apps, speed | ❌ |
| **Failover** | Health check status | ✅ Required | DR, active-passive | ❌ |
| **Geolocation** | User's country/continent | ✅ Per record | Compliance, localization | ❌ |
| **Geoproximity** | Geographic distance + bias | ✅ Per record | Regional coverage tuning | ✅ |
| **Multivalue** | Returns ≤8 healthy IPs | ✅ Per record | Basic redundancy | ❌ |
| **IP-Based** | Resolver CIDR block | ✅ Per record | Corporate/ISP routing | ❌ |

---

## 9. Common Pitfalls

### 1. Forgetting to update NS records at the registrar
Your Route 53 Hosted Zone is useless until you point your domain's NS records to the Route 53 name servers. This is the #1 cause of "why isn't DNS working?"

### 2. Using CNAME at the zone apex
```
❌ CNAME example.com → my-alb.elb.amazonaws.com  (invalid)
✅ A record (Alias) example.com → my-alb.elb.amazonaws.com  (correct)
```

### 3. Not creating a Default record in Geolocation routing
Users from unmatched locations get `NODATA` and can't reach your site.

### 4. Confusing Geolocation with Geoproximity
- **Geolocation:** Strict country/continent rules. User in Germany → Germany server, regardless of distance.
- **Geoproximity:** Distance-based. User in Germany → whichever server is physically closest.

### 5. Thinking Multivalue Answer replaces a load balancer
It doesn't. No sticky sessions, no connection draining, no SSL termination. Use an ALB for those.

### 6. High TTL during deployments
Setting TTL to 86400 (24 hours) during a deployment means DNS changes take up to 24 hours to propagate. Lower TTL before major changes, then raise it after.

### 7. Health checks not accounting for security groups
Route 53 health checkers have known IP ranges. If your security group blocks them, health checks always fail and Route 53 marks your endpoint unhealthy.

---

## 10. Real-World Architecture Example

### Scenario: Global SaaS Application

**Requirements:**
- Users in India → Mumbai servers
- Users in EU → Frankfurt servers
- Users in US → Virginia servers
- Automatic failover if a region goes down
- Canary deploy: 5% traffic to new version in US

**Architecture:**

```
                        ┌──────────────┐
                        │   Route 53   │
                        │  Latency +   │
                        │  Geolocation │
                        └──────┬───────┘
               ┌───────────────┼───────────────┐
               ▼               ▼               ▼
        [ap-south-1]     [eu-central-1]   [us-east-1]
         Mumbai ALB       Frankfurt ALB    Virginia ALB
               │                               │
               │                     ┌─────────┴─────────┐
               │                     ▼                   ▼
               │               [v1 servers 95%]   [v2 servers 5%]
               │               weighted routing    (canary)
               ▼
        RDS Multi-AZ
```

**DNS Records:**

```
# Latency record — India
api.example.com  A  Alias → ap-south-1-alb  Latency  ap-south-1  hc-india

# Latency record — EU
api.example.com  A  Alias → eu-central-1-alb  Latency  eu-central-1  hc-eu

# Latency record — US (points to weighted sub-policy)
api.example.com  A  Alias → us-east-1-alb  Latency  us-east-1  hc-us

# Weighted records behind us-east-1-alb
api-v1.us.internal  A  203.0.113.10  Weighted  95  hc-v1
api-v2.us.internal  A  203.0.113.20  Weighted  5   hc-v2
```

**Failover behavior:**

```
If ap-south-1 fails:
  → India users fail health check
  → Route 53 falls back to next-lowest-latency region (likely eu-central-1 or us-east-1)
  → Users experience slightly higher latency but service remains up
```

This architecture combines **Latency routing** (for global performance), **Weighted routing** (for canary deploys), and **Health checks** (for automatic failover) — demonstrating how routing policies can be layered for robust production systems.

---

## Quick Reference

```bash
# Create hosted zone
aws route53 create-hosted-zone \
  --name example.com \
  --caller-reference $(date +%s)

# List hosted zones
aws route53 list-hosted-zones

# List records in a hosted zone
aws route53 list-resource-record-sets \
  --hosted-zone-id Z1234567890ABC

# Check health check status
aws route53 get-health-check-status \
  --health-check-id abc123

# Test DNS resolution
dig A www.example.com
dig A www.example.com @ns-1234.awsdns-28.org   # query specific NS directly
nslookup www.example.com

# Check NS delegation
dig NS example.com +short
```

---

*Last updated: May 2026 | AWS Route 53 documentation: https://docs.aws.amazon.com/route53/*
