# Case Study 01: EC2 Fails to Register with AWS Systems Manager in a Private VPC (AWS)

##Summary
This case study server as a clear, easy-to-follow example and demonstration of AI-Human collaboration on system design and network debugging, highlighting its strengths and weak points, as well as shows my general understanding of cloud architecture. In this case, I troubleshoot a Systems Manager (SSM) registration failure for an EC2 instance deployed inside a Private VPC, designed to support ultra-secure, network-minimized Amazon Bedrock inference workloads on AWS. By design, the instance had no public IP and no exposed SSH ports, relying entirely on SSM for remote management and diagnostics. Despite correct IAM role attachment and VPC endpoint configuration, the instance failed to register with SSM due to a missing self-referencing inbound rule in its security group, which silently blocked traffic required for SSM agent functionality.

---

## Author Note
*** needs to be revised since I didn't actually inject updated aws doc** This case highlights more than a technical misconfiguration — it reveals how large language models can exhibit contextual blind spots when diagnosing layered, security-conscious architectures. The LLM initially misprioritized the root cause, not due to a lack of computational reasoning, but due to a framing bias influenced by incomplete system constraints. Once I injected the correct networking documentation and clarified the intended zero-public-access Bedrock design, the model pivoted toward accurate root cause analysis. This reinforces the importance of human-led contextual correction when collaborating with AI on advanced infrastructure troubleshooting.

---

## Problem Statement

My EC2 instance, bedrock-ec2, intentionally deployed with no public IP and no exposed SSH ports, failed to register as a managed node with AWS Systems Manager. While this did not inherently block the instance from interacting with other AWS services (such as Bedrock inference), the lack of SSM registration critically prevented remote diagnostics and secure automation workflows — which were essential given the architectural decision to forgo SSH entirely. The registration failure undermined the visibility required to confidently validate and monitor the Bedrock inference environment within this locked-down VPC.

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

     But the instance still failed to register with Systems Manager...
     
---

## LLM Misdiagnosis — Real Example of AI Framing Bias

As part of my diagnostic workflow, I involved GPT-4o to help interpret the failed ssm:SendCommand behavior:

Command Sent:

aws ssm send-command \
  --document-name "AWS-RunShellScript" \
  --targets "Key=tag:Name,Values=bedrock-ec2" \
  --parameters 'commands=["sudo systemctl restart amazon-ssm-agent"]' \
  --region us-east-1

The command returned a valid CommandId, but when querying invocation details:

aws ssm list-command-invocations \
  --command-id 19df72ff-436f-4385-b8a7-52a35c5b2bde \
  --details \
  --region us-east-1


Returned Response:

{
  "CommandInvocations": []
}


LLM Interpretation (Incorrect Diagnostic):

The model generated a plausible-sounding but incorrect conclusion:

|🔍 Meaning:
|	•	The command was successfully created
|	•	But no instance ever picked it up
|	•	This confirms: your EC2 instance is still not registered as a managed SSM node
|
|
|🚨 Final Confirmation
|
|All signs now point to one root issue:
|
|🔒 The SSM agent on your instance never successfully registered with Systems Manager, despite:
|
|	•	IAM role being correctly attached
|	•	SSM VPC endpoints being created
|	•	Quick Setup succeeding
|
|
|This could be due to:
|	•	A corrupted or unresponsive SSM agent
|	•	OS or AMI-level config drift
|	•	Timing issues during role attachment + boot


---

 
## Why This Interpretation Fell Short

GPT-4o’s early analysis hyper-focused on internal instance health, such as SSM agent corruption/failure to boot, AMI misconfiguration, or role timing conflicts. This framing bias neglected deeper network-layer constraints, despite the clear architectural constraints:

No public IP
No SSH access
Sole reliance on VPC endpoints and private networking

This oversight reinforced the incorrect assumption that the problem originated within the instance itself, rather than in its surrounding network configuration.

---


## AI-Human Collaboration Pivot

Despite the narrow initial focus, GPT-4o’s broader diagnostic checklist - after a few extra back-and-forth prompts - eventually recommended auditing the EC2 instance’s security group configuration within the AWS Console, listed as 1 of 8 recommended checks.


**My prompt**

"ec2-bedrock [my instance] still isn't showing up in AWS SSM Fleet manager, we need to identify every single factor preventing the instance as a systems-managed node."

**GPT-4o Suggested Diagnostic Checklist:**

"Given your EC2 instance isn’t showing as a managed node in Systems Manager, let's walk through the folllowing checks to ensure proper node registration".

|1️⃣ Confirm the SSM agent is installed, running, and up to date on the instance
|2️⃣ Double-check the attached IAM role includes AmazonSSMManagedInstanceCore
|3️⃣ Verify your VPC interface endpoints (ssm, ssmmessages, ec2messages) are active and reachable
|4️⃣ Ensure DNS resolution is enabled for the VPC and endpoint traffic
|5️⃣ Review route tables to confirm correct routing to VPC endpoints
|6️⃣ Inspect your instance’s security group rules, especially for both outbound and inbound rules.
|7️⃣ Check for overly restrictive NACLs (network ACLs) that could block traffic
|8️⃣ Review CloudWatch logs or system logs for SSM agent errors"

Once you've confirmed each item, I'll help you troubleshoot what's left step by step — we'll get this sorted. 

*Generated by GPT-4o on June 5, 2025*



## This output prompt became a key pivot point. ##


Through iterative human-led questioning, I had produced a list of possible failpoints. After reading all options, I realized I had never checked nor configured custom security group rules, so I investigated bullet point 6 further. Upon doing so, I discovered that the security group lacked an inbound rule permitting TCP 443 traffic from itself. an often-overlooked prerequisite for VPC endpoint communication in isolated environments.


## Root Cause

Although the IAM role and VPC endpoints were configured correctly, the EC2 instance’s security group lacked a self-referencing inbound rule allowing HTTPS traffic (TCP 443) from within the same group.

This silently blocked traffic from the EC2 instance to its associated VPC interface endpoints — despite all resources sharing the same security group.

The critical misunderstanding — shared by both human assumptions and the LLM’s initial framing — was believing that shared SG membership inherently permitted internal communication. In reality, a self-referencing inbound rule is explicitly required for intra-SG traffic within AWS security boundaries.

---

## Final Fix

Edited the EC2 security group to add an **inbound rule allowing TCP 443 from itself** (using the SG ID, not the CIDR block).

## Result

Instance successfully appeared in Systems Manager > Managed Instances.

---

## LLM Collaboration Reflection

Initially, the LLM focused on IAM trust relationships, SSM agent state, and endpoint availability. Once I introduced a linked AWS doc about instance visibility, it pivoted to emphasize network-level configuration.

This shift in focus — prompted by human-curated documentation — was a turning point. The combination of LLM-driven diagnostic depth and human-led contextual shifts led to identifying a subtle, low-level misconfiguration that could have been missed in a pure checklist approach.

---

## Lessons Learned

1. Technical Lessons

   - Shared security groups **do not guarantee** internal communication. Self-referencing inbound rules are required.
   - Interface endpoints marked “Available” can still be non-functional if security group or route conditions block traffic.
   - Successful SSM registration in a private VPC requires a complete stack:
      - Proper SG rules (inbound and outbound)  
      - IAM trust + permissions  
      - Valid DNS for endpoints  
      - SSM agent installation and reachability

2.  Human-AI collaboration Lessons
    
- Human-derived documentation-triggered reframing can dramatically improve LLM troubleshooting performance.

