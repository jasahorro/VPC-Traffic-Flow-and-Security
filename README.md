<img src="https://cdn.prod.website-files.com/677c400686e724409a5a7409/6790ad949cf622dc8dcd9fe4_nextwork-logo-leather.svg" alt="NextWork" width="300" />

# VPC Traffic Flow and Security

**Project Link:** [View Project](https://nextwork.ai/projects/fe5ed473-5874-57e1-a58f-d9b3d652330c)

**Author:** Maria Jasmin Ahorro  
**Email:** jasminahorro23@gmail.com

---

![Image](https://nextwork.ai/overjoyed_magenta_proud_yak/uploads/fe5ed473-5874-57e1-a58f-d9b3d652330c_92b0b0b4)

## Introducing Today's Project!

### What is Amazon VPC?

Amazon VPC (Virtual Private Cloud) is a foundational AWS networking service that lets you provision a logically isolated, custom virtual network within the AWS cloud. It provides complete control over your virtual networking environment, allowing you to define IP address ranges, configure subnets, manage route tables, and set up network gateways.

It is useful because it offers a secure, customizable, and scalable foundation for running your applications and hosting cloud resources. By isolating your AWS environment from other tenants, an Amazon VPC ensures that your workloads remain private and protected from unauthorized access.

Furthermore, a VPC allows you to divide your network into public and private subnets, giving you fine-grained control over exposure. You can deploy public-facing resources—like web servers—in public subnets with direct internet access, while placing sensitive databases and backend systems in private subnets shielded from external entry. Integrated security features like security groups and network ACLs add layered protection at both the resource and subnet levels.

### How I used Amazon VPC in this project

In today's project, I used Amazon VPC to build and secure a customized virtual network environment in AWS. I created a custom route table with a default route targeting an Internet Gateway (0.0.0.0/0) and associated it with my public subnet to enable external connectivity. To protect my network, I implemented a stateful NextWork Security Group with an inbound HTTP rule on port 80 to control access at the resource level, and deployed a stateless NextWork Network ACL to inspect and govern data packets entering and leaving at the subnet level.

### One thing I didn't expect in this project was...

One thing I didn't expect in this project was that custom Network ACLs default to denying all incoming and outgoing traffic until rules are explicitly added, whereas the default AWS Network ACL allows everything. I also didn't expect that security groups are stateful and operate at the instance level, while NACLs act as a stateless firewall for the entire subnet, requiring both to be configured carefully for complete network security.

### This project took me...

This project took me around 30 minutes to complete.

## Route tables

Route tables are a set of rules, called routes, that determine where network traffic from your subnets or gateways is directed. Within an Amazon VPC, every subnet must be linked to a route table, which essentially acts as a virtual router or GPS for your network—guiding data packets to their proper destinations inside or outside the cloud environment.

Each route in a table consists of a Destination (where the traffic wants to go) and a Target (the path or gateway it takes to get there). For instance, every custom route table automatically includes a default local route—such as 10.0.0.0/16 targeting local—which ensures that resources within the same VPC can easily talk to each other.

Route tables are also what determine whether a subnet is public or private. A subnet becomes public when its associated route table contains a default route targeting an Internet Gateway (written as 0.0.0.0/0 pointing to an igw-xxxxxxxx). This allows resources inside that subnet to send and receive traffic from the public internet.

Route tables are needed to make a subnet public because subnets in an Amazon VPC are isolated by default and cannot directly communicate with external networks. A route table contains the routing rules that direct outbound internet traffic (typically 0.0.0.0/0) to an attached Internet Gateway, enabling resources within that subnet to send and receive traffic to and from the public internet.

![Image](https://nextwork.ai/overjoyed_magenta_proud_yak/uploads/fe5ed473-5874-57e1-a58f-d9b3d652330c_0a07b191)

## Route destination and target

Routes are defined by their destination and target, which mean the destination specifies the IP address range where network traffic wants to go, while the target specifies the gateway, interface, or path through which that traffic is routed to reach its destination.

For instance, setting a destination of 0.0.0.0/0 with a target of an Internet Gateway (igw-xxxxxx) directs all internet-bound traffic out to the public internet, whereas a destination like 10.0.0.0/16 with a target of local keeps internal traffic within the VPC.

The route in my route table that directed internet-bound traffic to my internet gateway had a destination of 0.0.0.0/0 and a target of my Internet Gateway (NextWork IG).

![Image](https://nextwork.ai/overjoyed_magenta_proud_yak/uploads/fe5ed473-5874-57e1-a58f-d9b3d652330c_0a07b191)

## Security groups

Security groups are stateful, virtual firewalls that control inbound and outbound network traffic at the individual resource level—such as for an Amazon EC2 instance or database—within an Amazon VPC. Unlike Network ACLs, which operate at the subnet boundary, a security group attaches directly to a specific resource, offering granular access control.

Every rule within a security group defines permissible traffic based on three primary criteria: protocol (such as HTTP, TCP, or SSH), port range (such as port 80 or port 22), and source or destination IP address ranges. By default, custom security groups deny all incoming traffic until explicit inbound rules are added, while allowing all outbound traffic.

Because security groups are stateful, any rule that permits inbound traffic automatically allows response traffic back out to the requester, regardless of outbound rule settings. Conversely, response traffic for allowed outbound requests is automatically allowed back in. If a security group has no rule allowing a certain type of traffic, that traffic is blocked by default.

### Inbound vs Outbound rules

Inbound rules are a set of permissions within a security group or Network ACL that control what incoming network traffic is allowed to reach your resources.

I configured an inbound rule that allows HTTP traffic on port 80 from anywhere (0.0.0.0/0), enabling external web visitors to access resources deployed in my public subnet while blocking all other unauthorized incoming connections.

Outbound rules are a set of security permissions that control the outgoing network traffic sent from your cloud resources to external destinations, such as other resources in your VPC or the public internet.

By default, my security group's outbound rule allows all outbound traffic (0.0.0.0/0 on all protocols and ports), enabling attached resources to initiate communication with any external destination without restriction.

![Image](https://nextwork.ai/overjoyed_magenta_proud_yak/uploads/fe5ed473-5874-57e1-a58f-d9b3d652330c_92b0b0b4)

## Network ACLs

Network ACLs (Access Control Lists) are stateless, subnet-level virtual firewalls that control inbound and outbound traffic entering and leaving your subnets.

Unlike security groups—which operate at the individual resource level (like an EC2 instance) and are stateful—Network ACLs provide a broader layer of security across an entire subnet. They evaluate data packets against a numbered list of rules in order, allowing or denying traffic based on protocol, port range, and IP address. Default Network ACLs allow all inbound and outbound traffic, whereas custom Network ACLs deny all traffic until custom rules are added.

### Security groups vs. network ACLs

Security groups operate at the individual resource level (such as an EC2 instance) and are stateful, meaning return traffic is automatically allowed. In contrast, Network ACLs (NACLs) operate at the subnet level and are stateless, meaning both inbound and outbound traffic rules must be explicitly configured.

Additionally, security groups only support Allow rules, whereas Network ACLs process rules sequentially in numbered order and support both Allow and Deny rules.

## Default vs Custom Network ACLs

### Similar to security groups, network ACLs use inbound and outbound rules

By default, a network ACL's inbound and outbound rules will allow all inbound and outbound traffic (for the default Network ACL created by AWS). However, for any custom Network ACL that you create, the default inbound and outbound rules will deny all traffic until you explicitly add allow rules.

In contrast, a custom ACL’s inbound and outbound rules are automatically set to deny all traffic until you explicitly add rules to allow it.

![Image](https://nextwork.ai/overjoyed_magenta_proud_yak/uploads/fe5ed473-5874-57e1-a58f-d9b3d652330c_4faeb056)

---

*Built with [NextWork](https://nextwork.ai) - [View this project](https://nextwork.ai/projects/fe5ed473-5874-57e1-a58f-d9b3d652330c)*
