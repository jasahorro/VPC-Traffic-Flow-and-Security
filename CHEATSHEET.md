# VPC Traffic Flow & Security: Cheat Sheet

Quick review notes for the NextWork **VPC Traffic Flow and Security** project.
Full write-up: [README.md](README.md)

---

## 0. What is a VPC?

- AWS puts every resource you create into a **Region**. You can see your current Region in the **top-right corner** of the AWS Console.
  - Exception: **global resources** like IAM users and roles don't belong to a Region.
- A **VPC (Virtual Private Cloud)** is your own private, isolated network inside a Region.

### The city analogy 🏙️
| AWS | Analogy |
|---|---|
| **Region** | A country |
| **VPC** | Your own city inside that country |
| **Subnets** | Neighborhoods in your city |
| **Route tables / security** | Traffic rules and security measures |
| **EC2 instances, S3 buckets, etc.** | The buildings and things in your city |
| **Other AWS accounts' VPCs** | Other cities, fully isolated from yours |

**Key idea:** your VPC is isolated from everyone else's, so you have full privacy and control over its layout and rules.

---

## 0.5. What is an IPv4 CIDR?

- **IP address** = a street address for a resource in your city 🏠
- **CIDR block** = a whole block of IP addresses, like marking out a zone in your city 🗺️

### How to read a CIDR block
An IPv4 address has **32 bits** (4 parts × 8 bits each). The number after the slash (`/`) is how many bits are **fixed**. The remaining bits can **vary**, and those are the addresses you get to use.

> **The smaller the number after the slash, the BIGGER the block.**

| CIDR block | Fixed bits | Bits that vary | IP address range | # of addresses |
|---|---|---|---|---|
| `10.0.0.0/8` | 8 (`10`) | 24 | `10.0.0.0` – `10.255.255.255` | 2²⁴ = **16,777,216** |
| `10.0.0.0/16` | 16 (`10.0`) | 16 | `10.0.0.0` – `10.0.255.255` | 2¹⁶ = **65,536** |
| `10.0.0.0/24` | 24 (`10.0.0`) | 8 | `10.0.0.0` – `10.0.0.255` | 2⁸ = **256** |

### Quick math 🧮
```
number of addresses = 2 ^ (32 − number after the slash)
```
Example: `/24` → 2^(32 − 24) = 2^8 = 256

**Shortcut:** each `/8` covers one part of the address.
- `/8` → only the 1st part is fixed (`10.x.x.x`)
- `/16` → the first 2 parts are fixed (`10.0.x.x`)
- `/24` → the first 3 parts are fixed (`10.0.0.x`)

**In my projects:** VPC = `10.0.0.0/16` (the whole city), public subnet = `10.0.0.0/24` (one neighborhood inside it).

---

## The big picture

```
Internet
   │
Internet Gateway (IGW)         ← door between the VPC and the internet
   │
Route Table                    ← GPS: "where does this traffic go?"
   │
Network ACL (subnet level)     ← guard at the subnet gate (stateless)
   │
Subnet
   │
Security Group (resource level) ← guard at each resource's door (stateful)
   │
EC2 instance / resource
```

Traffic coming from the internet passes through **every** layer. If any one of them blocks it, the traffic doesn't get through.

---

## 1. Route tables

| Term | Meaning |
|---|---|
| **Route table** | A set of rules (routes) that decides where traffic from a subnet goes |
| **Destination** | Where the traffic wants to go (an IP range) |
| **Target** | The path or gateway it takes to get there |
| **Local route** | Added automatically, e.g. `10.0.0.0/16 → local`, so resources inside the VPC can talk to each other |

**Every subnet must be associated with a route table.**

### What makes a subnet public
A subnet is **public** when its route table has this route:

| Destination | Target |
|---|---|
| `0.0.0.0/0` | `igw-xxxxxxxx` (Internet Gateway) |

- `0.0.0.0/0` means "all IPv4 addresses", i.e. anywhere on the internet.
- Without this route, the subnet is private, even if an IGW is attached to the VPC.

### Steps I did
1. Created a custom route table in my VPC
2. Added the route `0.0.0.0/0 → NextWork IG`
3. Associated the route table with my **public subnet**

---

## 2. Security groups (SG)

- A virtual firewall attached to **individual resources** (EC2, RDS, and so on)
- **Stateful:** if a request is allowed in, the response is automatically allowed out (and the same in reverse)
- **Allow rules only.** You can't write a "deny" rule.
- Each rule = **protocol** + **port range** + **source/destination IP**

### Default behavior of a new SG
| Direction | Default |
|---|---|
| Inbound | ❌ Deny all (until you add rules) |
| Outbound | ✅ Allow all |

### The rule I added
| Type | Protocol | Port | Source |
|---|---|---|---|
| HTTP | TCP | 80 | `0.0.0.0/0` (anywhere) |

This lets anyone on the internet visit a web server in the subnet.

### Common ports to remember
| Port | Protocol |
|---|---|
| 22 | SSH |
| 80 | HTTP |
| 443 | HTTPS |
| 3389 | RDP (Windows remote desktop) |

---

## 3. Network ACLs (NACL)

- A virtual firewall for the **whole subnet**
- **Stateless:** return traffic is NOT automatically allowed, so you must write both inbound AND outbound rules
- Supports **Allow AND Deny** rules
- Rules are checked **in numbered order (lowest first)**. The first match wins.
- Every NACL ends with a `*` rule that denies anything not matched.

### Default vs custom NACL ⚠️ (the part that surprised me)
| NACL type | Inbound | Outbound |
|---|---|---|
| **Default** (made by AWS) | ✅ Allow all | ✅ Allow all |
| **Custom** (made by you) | ❌ Deny all | ❌ Deny all |

A new custom NACL blocks **everything** until you add allow rules.

---

## 4. Security group vs network ACL

| | Security Group | Network ACL |
|---|---|---|
| Works at | Resource level (EC2) | Subnet level |
| State | **Stateful** | **Stateless** |
| Rule types | Allow only | Allow + Deny |
| Rule order | All rules evaluated | Numbered, lowest first, first match wins |
| Return traffic | Auto-allowed | Must be explicitly allowed |
| Default (new/custom) | Inbound deny, outbound allow | Custom: deny all both ways |

**Memory trick:** **S**ecurity **G**roup = **S**tateful (it remembers the conversation). **NACL** = **N**o memory (stateless).

---

## 5. Inbound vs outbound

- **Inbound** = traffic coming **in** to your resource (e.g. a visitor loading your website)
- **Outbound** = traffic going **out** from your resource (e.g. your server downloading updates)

---

## Quick self-quiz

1. Which AWS resources don't belong to a Region? <br>→ Global resources like IAM users and roles
2. Which is bigger: `/16` or `/24`? <br>→ `/16` (smaller number = bigger block)
3. How many IP addresses are in a `/24`? A `/16`? <br>→ 256 / 65,536
4. What's the IP range of `10.0.0.0/16`? <br>→ `10.0.0.0` – `10.0.255.255`
5. What route makes a subnet public? <br>→ `0.0.0.0/0 → Internet Gateway`
6. Is a security group stateful or stateless? <br>→ Stateful
7. Is a NACL stateful or stateless? <br>→ Stateless
8. Which one can have Deny rules? <br>→ NACL only
9. What does a brand-new custom NACL allow? <br>→ Nothing. It denies all inbound and outbound.
10. What does the default NACL allow? <br>→ Everything
11. A new security group's default inbound/outbound? <br>→ Inbound: deny all · Outbound: allow all
12. What's the local route for? <br>→ Lets resources inside the VPC talk to each other
13. In what order are NACL rules evaluated? <br>→ By rule number, lowest first. The first match wins.
14. Port for HTTP? HTTPS? SSH? <br>→ 80 / 443 / 22

---

## Related projects
- Previous: **Build a Virtual Private Cloud** (VPC, subnets, internet gateway)
