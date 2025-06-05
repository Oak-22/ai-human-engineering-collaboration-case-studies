# Case Study 01: EC2 Fails to Register with AWS Systems Manager in a Private VPC

**Summary:**  
This case study documents the process of troubleshooting a Systems Manager (SSM) registration failure for an EC2 instance running in a private VPC, intended to support Amazon Bedrock inference workloads. The root cause was a missing self-referencing inbound rule in the EC2 instance’s security group, which prevented communication with the VPC interface endpoints for SSM — despite correct IAM role configuration and endpoint setup.

---

**Author Note:**  
This case is less about a rare technical edge case and more about how large language models require proactive context redirection. The LLM in this collaboration initially misprioritized the root cause not due to a lack of intelligence, but due to a framing bias. Once I introduced the right AWS documentation and reframed the constraints, the model pivoted effectively. This study highlights the importance of human intervention in guiding LLMs to the right diagnostic depth.

---

## Problem Statement

My EC2 instance, `bedrock-ec2`, failed to register as a managed node in AWS Systems Manager. This blocked automation and secure command execution — a critical part of my architecture for secure, private Amazon Bedrock inference.

---

## Investigative Steps

1. Verified IAM role trust relationship and attached policies:
   - `AmazonSSMManagedInstanceCore`
   - `AmazonSSMPatchAssociation`
   - `AmazonSSMManagedEC2InstanceDefaultPolicy`

2. Confirmed the EC2 instance used a modern AMI with a preinstalled and enabled SSM agent.

3. Confirmed the required VPC interface endpoints were created:
   - `com.amazonaws.us-east-1.ssm`
   - `com.amazonaws.us-east-1.ec2messages`
   - `com.amazonaws.us-east-1.ssmmessages`

4. Verified that both the EC2 instance and the interface endpoints used the same security group (`bedrock-private-link-sg`), which allowed outbound HTTPS (TCP 443).

5. Investigated route table configuration:
   - Confirmed route table was set to “main” and included standard routes.
   - Found no explicit route table associations with the interface endpoints.
   - Assumed this might be a blocking factor — but later proved irrelevant.

6. Performed several attempts to trigger registration:
   - Rebooted the instance
   - Issued `ssm:SendCommand` to restart the agent
   - Pulled and analyzed instance logs
   - Still failed to register with Systems Manager

---

## Root Cause

Although the IAM role and endpoints were configured correctly, the EC2 instance's **security group lacked an inbound rule allowing TCP 443 from itself**.

This blocked traffic from the EC2 instance to the VPC interface endpoints — **despite all resources sharing the same security group**.

The critical misunderstanding was assuming that sharing the same SG inherently permits communication. In reality, a self-referencing inbound rule is required for intra-SG communication.

---

## Final Fix

1. Edited the EC2 security group to add an **inbound rule allowing TCP 443 from itself** (using the SG ID, not the CIDR block).
2. Confirmed that interface endpoints were in the correct subnets.
3. Verified outbound rules and DNS resolution were in place.
4. Instance successfully appeared in Systems Manager > Managed Instances.

---

## LLM Collaboration Reflection

Initially, the LLM focused on IAM trust relationships, SSM agent state, and endpoint availability. Once I introduced a linked AWS doc about instance visibility, it pivoted to emphasize network-level configuration.

This shift in focus — prompted by human-curated documentation — was a turning point. The combination of LLM-driven diagnostic depth and human-led contextual shifts led to identifying a subtle, low-level misconfiguration that could have been missed in a pure checklist approach.

---

## Lessons Learned

- Shared security groups **do not guarantee** internal communication. Self-referencing inbound rules are required.
- Interface endpoints marked “Available” can still be non-functional if security group or route conditions block traffic.
- Documentation-triggered reframing can dramatically improve LLM troubleshooting performance.
- Successful SSM registration in a private VPC requires a complete stack:
  - IAM trust + permissions  
  - Valid DNS for endpoints  
  - Proper SG rules (inbound and outbound)  
  - SSM agent installation and reachability
