Transit Gateway — Two-Region GUI Step-by-Step (For Students)

Purpose: This is a clean, copy-ready instructor handout that walks students through creating a Transit Gateway (TGW) in two AWS regions, attaching one VPC per region, creating an inter-region TGW peering attachment, and configuring route tables so the two VPCs can communicate privately.


---

Assumptions & prerequisites

You have two existing VPCs in two different regions (example below uses Region A and Region B).

Example VPCs (use your actual VPC IDs/CIDRs):

VPC-A (Region A: ap-south-1) — CIDR 10.10.0.0/16

VPC-B (Region B: us-east-1) — CIDR 10.20.0.0/16



Non-overlapping CIDRs (critical).

At least one subnet in each VPC (recommended: one subnet in each AZ you expect attachments in).

You are in the same AWS account for both TGWs (steps below assume single account). Cross-account steps add an accept/ share step (not covered here).

IAM permissions for VPC/TGW actions (ec2:CreateTransitGateway, ec2:CreateTransitGatewayVpcAttachment, ec2:CreateTransitGatewayPeeringAttachment, ec2:AcceptTransitGatewayPeeringAttachment, ec2:CreateRoute, ec2:AssociateTransitGatewayRouteTable, etc.).



---

Quick architecture (one-line)

VPC-A (Region A) — attach → TGW-A — peering ↔ TGW-B — attach → VPC-B (Region B)


---

Step-by-step GUI instructions

> Tip: follow the steps in order. Use the AWS Region selector (top-right) to switch between Region A and Region B when instructed.



1) Prepare/confirm VPCs and subnets

1. AWS Console → Services → VPC → Your VPCs


2. Confirm VPC IDs and CIDRs (e.g., 10.10.0.0/16 and 10.20.0.0/16).


3. AWS Console → Subnets: ensure each VPC has at least one subnet. Note subnet IDs; you will select them for VPC attachment.



2) Create Transit Gateway in Region A

1. Switch Region (top-right) → Region A (e.g., ap-south-1).


2. AWS Console → Services → VPC → left menu → Transit Gateways.


3. Click Create transit gateway.


4. Fill form:

Name tag: tgw-regionA (or TGW-A)

Description: Transit Gateway for Region A — Lab (optional)

Amazon side ASN: leave default or set (e.g., 64512) — this matters only for BGP scenarios with Direct Connect/VPN.

Default Route Table Association: Enable (recommended for labs)

Default Route Table Propagation: Enable (recommended for labs)

Tags: optional (add Environment:lab)



5. Click Create transit gateway.


6. Wait until the TGW state shows available.



3) Create Transit Gateway in Region B

Repeat the same steps in Region B (switch region first):

1. Region → Region B (e.g., us-east-1).


2. VPC → Transit Gateways → Create transit gateway.


3. Name it tgw-regionB (or TGW-B) and use same recommended settings.


4. Create and wait until available.



4) Attach VPC-A to TGW-A

(Do this in Region A)

1. Region → Region A.


2. VPC Console → Transit Gateway Attachments (left menu).


3. Click Create transit gateway attachment.


4. Choose:

Transit gateway: select tgw-regionA.

Attachment type: VPC.

VPC: select VPC-A (your VPC in Region A).

Subnets: select one subnet (must be from the VPC) — choose at least one subnet per AZ you want connectivity from (select 1–2 subnets for the lab).

Tags: optional.



5. Click Create transit gateway attachment.


6. The attachment shows pending and should change to available (usually quick in same account).



5) Attach VPC-B to TGW-B

(Do this in Region B)

1. Region → Region B.


2. VPC → Transit Gateway Attachments → Create transit gateway attachment.


3. Select tgw-regionB, choose VPC attachment type, pick VPC-B, choose subnet(s), and Create.


4. Wait for status available.



6) Create Inter-Region Transit Gateway Peering (TGW-A ↔ TGW-B)

(You will create a peering attachment from Region A to Region B, then accept in Region B.)

In Region A:

1. Region → Region A.


2. VPC → Transit Gateway Peering Attachments (left menu) → Create transit gateway peering attachment.


3. Form fields:

Local transit gateway: select tgw-regionA.

Peer transit gateway: choose Other AWS Region and select Region B.

Peer transit gateway ID: paste the TGW ID of tgw-regionB (copy from TGW list in Region B).

Peer account ID: if same account, your account ID.

Name tag: tgw-peering-A-to-B.



4. Click Create transit gateway peering attachment.



In Region B (accept):

1. Switch to Region B.


2. VPC → Transit Gateway Peering Attachments → you will see an incoming peering with state pendingAcceptance.


3. Select it and click Accept.


4. After acceptance, both sides should show the peering available.



7) Configure Transit Gateway Route Tables (TGW route tables)

> Note: If you enabled default association/propagation at TGW creation, attachments may auto-associate. Still, you must add explicit routes for inter-region CIDRs.



On TGW-A (Region A):

1. Region → Region A → VPC → Transit Gateway Route Tables.


2. Select the route table used by your TGW attachments (Default or a custom one). Click Routes → Edit routes → Create route.


3. Destination: 10.20.0.0/16 (CIDR of VPC-B).


4. Target: select Transit gateway peering attachment and choose the peering attachment ID that points to TGW-B.


5. Save routes.



On TGW-B (Region B):

1. Region → Region B → VPC → Transit Gateway Route Tables.


2. Edit routes: add route 10.10.0.0/16 → Target = TGW peering attachment to TGW-A.



8) Configure VPC Route Tables (VPC → route via TGW attachment)

In VPC-A (Region A):

1. Region → Region A → VPC → Route Tables → select the route table used by the subnets that host your EC2 instances.


2. Click Routes → Edit routes → Add route.

Destination: 10.20.0.0/16 (VPC-B CIDR)

Target: select Transit Gateway and choose the tgw-regionA VPC attachment.



3. Save routes.



In VPC-B (Region B):

1. Region → Region B → VPC → Route Tables → choose route table for instances.


2. Add route: Destination 10.10.0.0/16 → Target = Transit Gateway attachment for tgw-regionB.


3. Save.



9) Security groups & NACLs: open ports for testing

Update Security Group(s) on test EC2 instances to allow inbound ICMP (ping) and/or SSH (TCP 22) from the remote VPC CIDR (e.g., allow 10.20.0.0/16 on VPC-A instances and 10.10.0.0/16 on VPC-B instances).

Network ACLs typically default to allow; ensure they are not blocking.


10) Test connectivity

1. Launch an EC2 instance in VPC-A (private IP 10.10.x.x) and one in VPC-B (10.20.x.x).


2. From EC2 in VPC-A, SSH to EC2 in VPC-B (private IP) or ping it (if ICMP allowed).

Linux example: ping -c 4 10.20.1.5

Traceroute: traceroute -n 10.20.1.5 (Linux) — you should see path go via TGW endpoints.



3. If connection fails, use the Troubleshooting checklist below.



Troubleshooting checklist (common gotchas)

Overlapping CIDRs — TGW won't route between overlapping VPC CIDRs. Ensure uniqueness.

Routes missing — Check both TGW route table and VPC route tables.

Peering not accepted — Verify peering state is available on both sides.

Security groups — Instance SGs must allow remote CIDR.

NACL — Might block traffic if modified from default.

Attachment not associated — Verify attachment is associated to the TGW route table (if you used a non-default RT).

Wrong subnets chosen for attachment — VPC attachment needs at least one valid subnet in the VPC.

Cross-account — If TGWs are in different accounts, ensure proper accept/permissions.


Cleanup steps (lab teardown)

1. In both regions: terminate test EC2 instances.


2. In Region B: go to Transit Gateway Peering Attachments → select peering → Delete.


3. Delete Transit Gateway Attachments from both regions (Transit Gateway Attachments → select → Delete).


4. Delete Transit Gateways (VPC → Transit Gateways → select → Delete).


5. Remove routes from VPC route tables.



Instructor notes / talking points

Show how TGW solves transitive routing: highlight difference from peering.

Emphasize cost (TGW hourly + data processing) — advise students to delete lab resources.

Discuss scaling: once the TGW hub is created, adding new VPCs is just an attachment + route.


Quick cheat-sheet (single page)

1. Create TGW in Region A & Region B.


2. Attach VPC-A → TGW-A; Attach VPC-B → TGW-B.


3. Create TGW peering from A→B and Accept in B.


4. Add TGW route: VPC-B_CIDR → TGW-peering on TGW-A; VPC-A_CIDR → TGW-peering on TGW-B.


5. Add VPC route: remote CIDR → Transit Gateway attachment.


6. Update SGs/NACLs and test.




---

If you want, I can also:

Convert this to a one-slide printable PDF for students, or

Produce screenshots for each GUI step (I can outline which console pages to screenshot during the demo), or

Provide a short lab exercise with exact EC2 instance types, sample private IPs, and step verifications.



---

Document prepared for classroom use — ready to distribute to students.



Done — I created a step-by-step GUI handout titled "Transit Gateway — Two-Region GUI Step-by-Step (For Students)" on the canvas. Open it to copy, print, or share with your students.

Want a PDF copy or a one-slide cheat-sheet next?

