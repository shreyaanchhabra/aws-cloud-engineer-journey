# Week 5: Networking, VPC Deep Dive

I built a custom VPC by hand in the console, then used a bastion host to reach servers that have no public IP. The diagram is in `vpc-diagram.png`.

## What I built

| Piece | Setting |
|---|---|
| VPC | `training-vpc`, `10.0.0.0/16`, us-east-1 |
| Public subnets | 2 (one per Availability Zone): `10.0.0.0/20`, `10.0.16.0/20` |
| Private subnets | 2 (one per Availability Zone): `10.0.128.0/20`, `10.0.144.0/20` |
| Internet Gateway | 1, attached to the VPC |
| NAT Gateway | 1, in a public subnet, with an Elastic IP |
| Bastion host | t3.micro in a public subnet, public IP `3.226.248.139`, private IP `10.0.2.154` |
| Private servers | Two t3.micro instances in the private subnets (`10.0.130.246`, `10.0.150.78`), no public IP |

## Public vs private subnets

A subnet is public only because its route table sends `0.0.0.0/0` to the Internet Gateway. A private subnet's route table sends `0.0.0.0/0` to the NAT Gateway instead. Routing is explicit: a subnet gets only the routes of the route table it is associated with, so "private" is not something a subnet inherits from its name.

## Traffic path: laptop to private instance

1. **Laptop to bastion.** I load my key into my SSH agent (`ssh-add`) and run `ssh -A ec2-user@3.226.248.139`. The packet leaves my laptop and crosses the internet.
2. **Internet Gateway.** It enters the VPC through the Internet Gateway. The bastion's public subnet has a route to the gateway, so the traffic can reach it.
3. **Bastion security group.** `bastion-sg` allows SSH (port 22) only from my own IP, so only my laptop gets in.
4. **Bastion to private server.** From the bastion I run `ssh ec2-user@<private-ip>`. This hop stays inside the VPC, using the private IP, and never touches the internet. The private servers' security group allows SSH only from `bastion-sg` (a security group, not an IP address), so nothing else can SSH in.
5. **Agent forwarding.** `-A` lets the bastion use my key through my laptop's agent, so the `.pem` file is never copied onto the bastion.

The private servers have no public IP, so nothing on the internet can start a connection to them. Reaching them requires going through the bastion.

## Outbound traffic from private servers

The private servers still need the internet for things like `dnf install` and `pip install`. Their route table sends `0.0.0.0/0` to the NAT Gateway, which sits in a public subnet. The NAT Gateway forwards the traffic out through the Internet Gateway using its Elastic IP, and return traffic comes back the same way. This is outbound only: the internet cannot start new connections to the private servers.

I used the same pattern in Week 6: the Auto Scaling group's servers live in the private subnets, and I reached them through this bastion to run the CPU stress test.

## Why the NAT Gateway is in only one AZ

I picked "NAT gateways: In 1 AZ" to keep cost down. The tradeoff is that if that AZ fails, the private subnet in the other AZ loses outbound internet access. A production setup would use one NAT Gateway per AZ.

## Cost note

A NAT Gateway bills about $0.045 per hour even when idle, and so does the Elastic IP attached to it when unattached. I deleted the NAT Gateway, released the Elastic IP, then deleted the VPC at the end of the week.
