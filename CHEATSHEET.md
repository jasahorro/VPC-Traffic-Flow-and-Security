# VPC Traffic Flow & Security: Cheat Sheet

Quick review notes for the NextWork **VPC Traffic Flow and Security** project.
Full write-up: [README.md](README.md)

**Contents**
1. [What is a VPC?](#1-what-is-a-vpc)
2. [IPv4 CIDR blocks](#2-ipv4-cidr-blocks)
3. [Subnets & auto-assign public IPv4](#3-subnets--auto-assign-public-ipv4)
4. [Internet gateways](#4-internet-gateways)
5. [Route tables](#5-route-tables)
6. [Security groups](#6-security-groups)
7. [Protocols & port numbers](#7-protocols--port-numbers)
8. [Inbound vs outbound rules](#8-inbound-vs-outbound-rules)
9. [Network ACLs](#9-network-acls)
10. [Security groups vs network ACLs](#10-security-groups-vs-network-acls)
11. [The big picture: a request's journey](#11-the-big-picture-a-requests-journey)
12. [Quick self-quiz](#quick-self-quiz)

---

## The city analogy 🏙️ (used throughout this project)

| AWS | Analogy |
|---|---|
| **Region** | A country |
| **VPC** | Your own city inside that country |
| **CIDR block** | A zone or area of the city |
| **IP address** | A street address |
| **Subnet** | A neighborhood |
| **Resource (EC2, etc.)** | A building |
| **Route table** | A GPS |
| **Network ACL** | Traffic cops at the neighborhood's entry and exit points |
| **Security group** | A security guard at each building's entrance |
| **Protocols** | Vehicles (buses, trucks, mail carriers) carrying data |
| **Port numbers** | Doors on a building |
| **Other AWS accounts' VPCs** | Other cities, fully isolated from yours |

---

## 1. What is a VPC?

- AWS puts every resource you create into a **Region**. You can see your current Region in the **top-right corner** of the AWS Console.
  - Exception: **global resources** like IAM users and roles don't belong to a Region.
- A **VPC (Virtual Private Cloud)** is your own private, isolated network inside a Region. You design its neighborhoods (subnets), traffic rules, and security.
- **Key idea:** your VPC is isolated from everyone else's, so you have full privacy and control over its layout and rules.

---

## 2. IPv4 CIDR blocks

- **IP address** = a street address for a resource 🏠
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

## 3. Subnets & auto-assign public IPv4

- **Subnet** = a neighborhood inside your VPC, with its own smaller CIDR block.
- **Auto-assign public IPv4 address** (a subnet setting):
  - Any EC2 instance launched in that subnet **automatically gets a public IP address**.
  - This makes the instance reachable from the internet without assigning an IP by hand, which saves a lot of time.

---

## 4. Internet gateways

- **Attaching an internet gateway (IGW) to a VPC** means resources in the VPC can now access the internet.
- EC2 instances **with public IPs** become reachable by users, so any app you host on them becomes public too.

> ⚠️ Attaching the IGW alone isn't enough. The subnet's **route table** also needs a route to it (see section 5).

### For an EC2 instance to be reachable from the internet, you need ALL of these:
1. ✅ An internet gateway **attached** to the VPC
2. ✅ A route `0.0.0.0/0 → igw-xxxx` in the subnet's **route table**
3. ✅ A **public IP** on the instance (e.g. from auto-assign public IPv4)
4. ✅ A **network ACL** that allows the traffic
5. ✅ A **security group** that allows the traffic (e.g. HTTP on port 80)

---

## 5. Route tables

- **Route table** = a **GPS** 🧭 for the resources in your subnet. It's a table of rules (called **routes**) that decide where network traffic goes.
- **Every subnet must be linked to a route table.** It tells the subnet's traffic where to go to send and receive data.
  - Example: a web server (EC2) hosting a website needs a route table that knows how to direct traffic to and from it.

### Destination vs target
| Column | Meaning | Analogy |
|---|---|---|
| **Destination** | The IP address range the traffic wants to reach | Where you want to go |
| **Target** | The path the traffic takes to get there | The road you take |

### Common targets
| Target | Meaning |
|---|---|
| `local` | Traffic **stays inside the VPC**, so resources can talk to each other |
| `igw-xxxxxx` | Traffic goes **to the internet** through the internet gateway |

### My route table
| Destination | Target | What it does |
|---|---|---|
| `10.0.0.0/16` | `local` | Traffic for anything inside my VPC stays inside (added automatically) |
| `0.0.0.0/0` | `igw-xxxx` (NextWork IG) | Everything else goes to the internet |

### Why `0.0.0.0/0`?
- `0.0.0.0/0` = **all IPv4 addresses**.
- It creates a **default route**: any traffic that doesn't match a more specific route goes here.
- In my table, any traffic **not** bound for `10.0.0.0/16` (my VPC) is sent to the internet gateway, which forwards it to the internet.

### 🧠 Extra for experts: which route wins?
Routes are checked from **most specific** (bigger number after the slash) to **least specific** (`/0`).
- Destination is inside `10.0.0.0/16`? → `local` (stays in the VPC)
- Otherwise → `0.0.0.0/0` (internet gateway)

### Internet gateway + route table = public subnet
> A subnet becomes **public** when its route table has a route that sends internet-bound traffic (`0.0.0.0/0`) to an **internet gateway**.
> Without that route, the subnet is private, even if an IGW is attached to the VPC.

### Steps I did
1. Created a custom route table in my VPC
2. Added the route `0.0.0.0/0 → NextWork IG`
3. Associated the route table with my **public subnet**

---

## 6. Security groups

- **Security group (SG)** = a **security guard** 💂 at the entrance of each **building** (resource).
- SGs attach to **individual resources**, **not** to a VPC or subnet.
- **Every resource must have a security group.** If you don't pick one, AWS uses the **default security group** it created with your VPC.
- SGs check what comes in and out, based on **IP address, protocol and port number**.
- **Stateful:** if a request is allowed in, the response is automatically allowed out (and the same in reverse).
- **Allow rules only.** You can't write a "deny" rule.

### Default behavior of a new SG
| Direction | Default |
|---|---|
| Inbound | ❌ Deny all (until you add rules) |
| Outbound | ✅ **Allow all**: resources can send data to any IP, whether in your VPC, in other VPCs (with permissions), or on the internet |

> 💡 That's why I only had to create an **inbound** rule. Outbound was already open.

### The rule I added
| Type | Protocol | Port | Source |
|---|---|---|---|
| HTTP | TCP | 80 | `0.0.0.0/0` (anywhere) |

This lets anyone on the internet visit a web server in the subnet.

---

## 7. Protocols & port numbers

### Protocols = vehicles 🚌
Protocols are rules for how data moves across the internet. Each one is built for a specific kind of task.

| Protocol | Full name | What it does | Analogy |
|---|---|---|---|
| **HTTP** | Hypertext Transfer Protocol | Sends web pages to your browser | 🚌 A bus carrying passengers |
| **SMTP** | Simple Mail Transfer Protocol | Sends emails | 📬 A mail carrier |
| **FTP** | File Transfer Protocol | Moves large files between computers (e.g. uploading website files to a server) | 🚚 A delivery truck |
| **SSH** | Secure Shell | Encrypted, private access to manage a server (e.g. EC2) | 🚗 A secure private car |

### Port numbers = doors 🚪
- Each port is a "door" on a server for a specific protocol.
- Ports tell the server **which application** should handle incoming data. Without them, an EC2 instance wouldn't know where to send each piece of data.

| Port | Protocol |
|---|---|
| **21** | FTP |
| **22** | SSH |
| **25** | SMTP |
| **80** | HTTP |
| **443** | HTTPS (secure HTTP) |
| **3389** | RDP (Windows remote desktop) |

---

## 8. Inbound vs outbound rules

| | Inbound ⬇️ | Outbound ⬆️ |
|---|---|---|
| Controls | Data **entering** your resources | Data your resources **send out** |
| Examples | Visitors to your website, form submissions | Your server requesting data from another service, your app sending an email notification |
| Why it matters | Lets users access your public website | Manages how your server talks to the rest of the internet |

---

## 9. Network ACLs

- **Network ACL (NACL)** = **traffic cops** 👮 at every entry and exit point of your **subnet**. They check each data packet against a table of rules before letting it through.
- **Stateless:** return traffic is NOT automatically allowed, so you must write both inbound AND outbound rules.
- Supports **Allow AND Deny** rules.
- Rules are checked **in numbered order (lowest first)**. The first match wins.

### What's a data packet? 📦
- "Traffic" is really **data packets**.
- Each packet carries a **piece** of the data being sent, such as part of a user's request, part of a web page, or part of a file.
- Example: when someone types your website's URL and presses Enter, the inbound traffic reaching your EC2 instance is data packets carrying that request.
- NACLs inspect these packets as they **enter and exit subnets**, so only authorized data gets through.

### Default NACL rules
| Rule # | Type | Source / Destination | Allow / Deny |
|---|---|---|---|
| **100** (Inbound) | All traffic | `0.0.0.0/0` | ✅ Allow |
| **\*** (Inbound) | All traffic | `0.0.0.0/0` | ❌ Deny |
| **100** (Outbound) | All traffic | `0.0.0.0/0` | ✅ Allow |
| **\*** (Outbound) | All traffic | `0.0.0.0/0` | ❌ Deny |

- **Rule 100** allows all traffic in and out of the subnet.
- The **`*` rule** is a **catch-all** for anything that didn't match an earlier rule. In the default NACL, Rule 100 already allows everything, so `*` never actually applies.

### Default vs custom NACL ⚠️ (the part that surprised me)
| NACL type | Inbound | Outbound |
|---|---|---|
| **Default** (made by AWS) | ✅ Allow all | ✅ Allow all |
| **Custom** (made by you) | ❌ Deny all | ❌ Deny all |

A new custom NACL has **only** the `*` deny rule, so it blocks **everything** until you add allow rules.

---

## 10. Security groups vs network ACLs

| | Security Group | Network ACL |
|---|---|---|
| Analogy | Guard at each building's entrance | Traffic cops at the neighborhood's entry points |
| Works at | **Resource** level (EC2) | **Subnet** level |
| Control | **Granular**: ports and protocols per resource | **Broad**: rules for the whole subnet (e.g. block an IP range, deny certain ports) |
| State | **Stateful** | **Stateless** |
| Rule types | Allow only | Allow + Deny |
| Rule order | All rules evaluated | Numbered, lowest first, first match wins |
| Return traffic | Auto-allowed | Must be explicitly allowed |
| Default (new/custom) | Inbound deny, outbound allow | Custom: deny all both ways |

**Memory trick:** **S**ecurity **G**roup = **S**tateful (it remembers the conversation). **NACL** = **N**o memory (stateless).

### Why use both? 🛡️🛡️
It's a security best practice called **layered security**:
- **NACLs** set **broad** restrictions at the subnet level.
- **Security groups** set **specific** limits at the resource level.
- Traffic has to pass **both checks**, which reduces the chance of unwanted access.

---

## 11. The big picture: a request's journey

What happens when someone visits a website hosted on my EC2 instance:

```
 💻 User's browser
      │  ⬇️ request            ⬆️ response
 🚪 Internet Gateway   (NextWork IG)
      │
 🌐 VPC                (NextWork VPC)
      │
 🚏 Route Table        (NextWork route table)   → local route
      │
 📋 Network ACL        (Rule 100: allow all from 0.0.0.0/0)
      │
 🥅 Public Subnet      (Public 1)
      │
 👮 Security Group     (NextWork Security Group: allow HTTP port 80 from 0.0.0.0/0)
      │
 🖥️ EC2 Instance       (web server builds the response)
```

### Step by step ⬇️ (the request going in)
| # | Stop | What happens |
|---|---|---|
| 1 | 💻 **Client/User** | The user types the website URL into their browser and presses Enter |
| 2 | 🚪 **Internet Gateway** | The request travels over the internet and reaches **NextWork IG** |
| 3 | 🌐 **VPC** | The IGW forwards the request into the VPC it's attached to, **NextWork VPC** |
| 4 | 🚏 **Route Table** | The public subnet's route table directs the request toward the EC2 instance using the **local** route |
| 5 | 📋 **Network ACL** | The request is checked at the subnet border. **Inbound Rule 100** allows traffic from anywhere (`0.0.0.0/0`), so it passes ✅ |
| 6 | 🥅 **Public Subnet** | The request enters **Public 1** and heads to the EC2 instance |
| 7 | 👮 **Security Group** | **NextWork Security Group** has an inbound rule allowing **HTTP (port 80)** from `0.0.0.0/0`, so it passes ✅ |
| 8 | 🖥️ **EC2 Instance** | The web server processes the request and prepares the response |

### The trip back ⬆️ (the response going out)
| # | What happens |
|---|---|
| 9 | The website content travels back the **same way in reverse**: Security Group → Public Subnet → Network ACL → Route Table (the `0.0.0.0/0 → IGW` route this time, since the user is out on the internet) → VPC → Internet Gateway → the user's browser 🎉 |

> 🧠 **Stateful vs stateless on the way back:**
> - **Security group:** the response is let out **automatically** because it's stateful.
> - **Network ACL:** the response needs an **outbound rule** that allows it because it's stateless. In the default NACL, **Outbound Rule 100** allows all.

**If any stop blocks the traffic, the website won't load.** When troubleshooting, walk through each stop in this order.

---

## Quick self-quiz

1. Which AWS resources don't belong to a Region? <br>→ Global resources like IAM users and roles
2. Which is bigger: `/16` or `/24`? <br>→ `/16` (smaller number = bigger block)
3. How many IP addresses are in a `/24`? A `/16`? <br>→ 256 / 65,536
4. What's the IP range of `10.0.0.0/16`? <br>→ `10.0.0.0` – `10.0.255.255`
5. What does "auto-assign public IPv4 address" do? <br>→ Gives every EC2 instance launched in the subnet a public IP automatically
6. What does attaching an internet gateway do? <br>→ Lets resources in the VPC access the internet, and makes instances with public IPs reachable
7. What route makes a subnet public? <br>→ `0.0.0.0/0 → Internet Gateway`
8. In a route table, what's the difference between destination and target? <br>→ Destination = where the traffic wants to go. Target = the path it takes to get there.
9. What does the target `local` mean? <br>→ The traffic stays inside the VPC
10. What does `0.0.0.0/0` mean? <br>→ All IPv4 addresses (the default route)
11. If traffic matches both `10.0.0.0/16` and `0.0.0.0/0`, which route wins? <br>→ `10.0.0.0/16`, because the most specific route wins
12. What do security groups attach to? <br>→ Individual resources, not VPCs or subnets
13. What happens if you launch a resource without choosing a security group? <br>→ It uses the VPC's default security group
14. Is a security group stateful or stateless? <br>→ Stateful
15. Is a NACL stateful or stateless? <br>→ Stateless
16. Which one can have Deny rules? <br>→ NACL only
17. A new security group's default inbound/outbound? <br>→ Inbound: deny all · Outbound: allow all
18. Why did I only need an inbound rule on my security group? <br>→ Security groups allow all outbound traffic by default
19. Give an example of inbound and outbound traffic. <br>→ Inbound: a visitor loading your site. Outbound: your app sending an email notification.
20. What is a data packet? <br>→ A small piece of the data being sent across a network
21. In the default NACL, what do Rule 100 and the `*` rule do? <br>→ Rule 100 allows all traffic. `*` is a catch-all deny for anything that didn't match an earlier rule.
22. What does a brand-new custom NACL allow? <br>→ Nothing. It denies all inbound and outbound.
23. In what order are NACL rules evaluated? <br>→ By rule number, lowest first. The first match wins.
24. Why use both NACLs and security groups? <br>→ Layered security: broad rules at the subnet level plus specific rules at the resource level
25. Which protocol is used for: web pages? Email? Big file transfers? Secure server access? <br>→ HTTP / SMTP / FTP / SSH
26. Ports for FTP, SSH, SMTP, HTTP, HTTPS? <br>→ 21 / 22 / 25 / 80 / 443
27. Name the stops a website request passes through, in order. <br>→ User → Internet Gateway → VPC → Route Table → Network ACL → Public Subnet → Security Group → EC2 instance
28. On the way back, which layer lets the response out automatically, and which needs an outbound rule? <br>→ The security group is automatic (stateful). The network ACL needs an outbound rule (stateless).

---

## Related projects
- Previous: **Build a Virtual Private Cloud** (VPC, subnets, internet gateway)
