# VPC Traffic Flow & Security: Cheat Sheet

Quick review notes for the NextWork **VPC Traffic Flow and Security** project.
Full write-up: [README.md](README.md)

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

1. What route makes a subnet public? <br>→ `0.0.0.0/0 → Internet Gateway`
2. Is a security group stateful or stateless? <br>→ Stateful
3. Is a NACL stateful or stateless? <br>→ Stateless
4. Which one can have Deny rules? <br>→ NACL only
5. What does a brand-new custom NACL allow? <br>→ Nothing. It denies all inbound and outbound.
6. What does the default NACL allow? <br>→ Everything
7. A new security group's default inbound/outbound? <br>→ Inbound: deny all · Outbound: allow all
8. What's the local route for? <br>→ Lets resources inside the VPC talk to each other
9. In what order are NACL rules evaluated? <br>→ By rule number, lowest first. The first match wins.
10. Port for HTTP? HTTPS? SSH? <br>→ 80 / 443 / 22

---

## Related projects
- Previous: **Build a Virtual Private Cloud** (VPC, subnets, internet gateway)
