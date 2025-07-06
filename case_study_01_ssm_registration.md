# Case Study 01: EC2 Fails to Register with AWS Systems Manager in a Private VPC (AWS)

## Summary
This case study documents troubleshooting an AWS Systems Manager (SSM) registration failure for an EC2 instance deployed inside a Private VPC, designed to support ultra-secure, network-minimized Amazon Bedrock inference workloads. By design, the instance had no public IP and no exposed SSH ports, relying entirely on SSM for remote management and diagnostics. Despite correct IAM role attachment and VPC interface endpoint configuration, the instance failed to register with SSM due to a missing self-referencing inbound rule in the instance's security group. This rule is required to enable traffic between the EC2 instance and VPC Interface Endpoints that share the same security group.

This case reflects how large language models can misprioritize root cause analysis when underlying architectural constraints are not properly framed. In this example, the LLM (GPT-4o) initially steered the troubleshooting process toward internal instance health, agent state, and IAM misconfigurations—despite clear indicators that this environment relied entirely on private networking and VPC Endpoints for communication. Once I provided targeted AWS documentation and clarified the design—zero public IP, no SSH exposure, sole reliance on VPC Interface Endpoints—the model adjusted its reasoning. However, the critical discovery came not from a direct AI insight, but from an LLM-generated diagnostic checklist that prompted me to manually inspect security group rules. This reinforced that human-led context correction remains essential in AI-assisted infrastructure debugging.

---

## Problem Statement

My EC2 instance bedrock-ec2, intentionally deployed with no public IP and no exposed SSH ports, failed to register as a managed node with AWS Systems Manager. While this did not inherently block the instance from interacting with AWS Bedrock (inference workloads functioned), the lack of SSM registration prevented secure remote diagnostics and automation workflows—both essential given the intentional restriction of SSH access.

---

## Investigative Steps

1. Verified IAM role trust relationship and attached policies:
   - `AmazonSSMManagedInstanceCore`
   - `AmazonSSMPatchAssociation`
   - `AmazonSSMManagedEC2InstanceDefaultPolicy`

2. Confirmed the EC2 instance used a modern AMI with a preinstalled and enabled SSM agent.

3. Validated VPC Interface Endpoints:
   - `com.amazonaws.us-east-1.ssm`
   - `com.amazonaws.us-east-1.ec2messages`
   - `com.amazonaws.us-east-1.ssmmessages`

4. Verified that both the EC2 instance and the interface endpoints used the same security group (`bedrock-privatelink-sg`), which allowed outbound HTTPS (TCP 443).

5. Checked Route Table and DNS Validation:
   - Confirmed route table was set to “main” and included standard routes.
   - DNS resolutional functional. 

6. Performed several attempts to trigger registration:
   - Rebooted the instance
   - Issued `ssm:SendCommand` to restart the agent
   - Pulled and analyzed instance logs

     But the instance still failed to register with Systems Manager...
     
---

## LLM Misdiagnosis — Real Example of AI Framing Bias

As part of my diagnostic workflow, I involved GPT-4o to help interpret the failed ssm:SendCommand behavior:

**Command Sent:**

aws ssm send-command \
  --document-name "AWS-RunShellScript" \
  --targets "Key=tag:Name,Values=bedrock-ec2" \
  --parameters 'commands=["sudo systemctl restart amazon-ssm-agent"]' \
  --region us-east-1

**The command returned a valid CommandId, but when querying invocation details:**

aws ssm list-command-invocations \
  --command-id 19df72ff-436f-4385-b8a7-52a35c5b2bde \
  --details \
  --region us-east-1


**Returned Response:**

{
  "CommandInvocations": []
}



**LLM Interpretation (Incorrect Diagnostic):**


The model proceeded to generate a plausible-sounding but incorrect conclusion as quoted below:


".... Meaning:
	•	The command was successfully created
	•	But no instance ever picked it up
	•	This confirms: your EC2 instance is still not registered as a managed SSM node


Final Confirmation

All signs now point to one root issue:

The SSM agent on your instance never successfully registered with Systems Manager, despite:

   •	IAM role being correctly attached
	•	SSM VPC endpoints being created
	•	Quick Setup succeeding


This could be due to:
	•	A corrupted or unresponsive SSM agent
	•	OS or AMI-level config drift
	•	Timing issues during role attachment + boot"


---

 
## Why This Interpretation Fell Short

GPT-4o’s analysis hyper-focused on internal instance health, such as SSM agent corruption/failure to boot, AMI misconfiguration, or role timing conflicts. This framing bias neglected deeper network-layer constraints, despite the clear architectural constraints:

No public IP
No SSH access
Sole reliance on VPC endpoints and private networking

This oversight reinforced the incorrect assumption that the problem originated within the instance itself, rather than in its surrounding network configuration.

---


## AI-Human Collaboration Pivot: Rewinding the Problem Space to a Known-Good State

The LLM exhibited a familiar diagnostic flaw I’ve seen repeatedly. <u>*Once a subtle misstep occurs in the reasoning process (often unnoticed by the user), the model tends to recursively tunnel deeper along that incorrect path. With each additional prompt in the same chat session, the model compounds its framing bias, drifting further from the root issue.*</u>

As the conversation lengthens and token limits are approached, essential technical context is degraded or forgotten altogether. At this stage, users must intervene to effectively "rewind" the LLM's problem space to a stable, known-good state. This is comparable to restoring a database from a snapshot or replaying a write-ahead log (WAL) to recover from corruption. Unlike databases, LLMs don’t persist conversational checkpoints inherently so this reset relies on human intervention, often with nuance. 

In low-complexity scenarios, a targeted reframing prompt can restore diagnostic alignment. In more entrenched sessions, starting a fresh chat becomes necessary. When doing so, I recommend prompting the LLM to summarize the prior conversation, preserving relevant system context while discarding compounding errors.

In this case, I strategically reset the diagnostic trajectory by requesting the LLM identify all potential blockers to Systems Manager registration. This guided the model to recommend auditing the EC2 instance’s security group settings within the AWS Console, listed as one of eight suggested checks. That recommendation became the pivotal clue that ultimately exposed the missing self-referencing inbound rule.


---


## Rewind Prompt

"ec2-bedrock [my instance] still isn't showing up in AWS SSM Fleet manager. Review the chat, review what factors we've confirmed are in working order and compare that to every possible factor preventing the instance's registration, and I will verify each factor in the checklist one at a time"

**GPT-4o Suggested Diagnostic Checklist:**

"That's a great idea! Let's approach this methodically as a true system's architect would. Given your EC2 instance isn’t showing as a managed node in Systems Manager, let's walk through the folllowing checks to ensure proper node registration.

1️Confirm the SSM agent is installed, running, and up to date on the instance
2️Double-check the attached IAM role includes AmazonSSMManagedInstanceCore
3️Verify your VPC interface endpoints (ssm, ssmmessages, ec2messages) are active and reachable
4️Ensure DNS resolution is enabled for the VPC and endpoint traffic
5️Review route tables to confirm correct routing to VPC endpoints
6️Inspect your instance’s security group rules, especially for both outbound and inbound rules.
7 Check for overly restrictive NACLs (network ACLs) that could block traffic
8️Review CloudWatch logs or system logs for SSM agent errors"

Once you've confirmed each item, I'll help you troubleshoot what's left step by step — we'll get this sorted."

*Generated by GPT-4o on June 5, 2025*


**This rewound-state output became a key pivot point.**


After restoring the chat's mental state by asking it to review our progress so far carefully and through iterative human-led questioning, I had produced a list of possible failpoints. After reading all options, I realized I had never checked nor configured custom security group rules, so I investigated bullet point 6 further. Upon doing so, I discovered that the security group only contained a default invound lacked an inbound rule permitting TCP 443 traffic from itself. an often-overlooked prerequisite for VPC endpoint communication in isolated environments.

---

## Root Cause

Although the IAM role and VPC endpoints were configured correctly, the EC2 instance’s security group lacked a self-referencing inbound rule allowing HTTPS traffic (TCP 443) from within the same group.

This silently blocked traffic from the EC2 instance to its associated VPC interface endpoints — despite all resources sharing the same security group.

The critical misunderstanding — shared by both human assumptions and the LLM’s initial framing — was believing that shared SG membership inherently permitted internal communication. In reality, a self-referencing inbound rule is explicitly required for intra-SG traffic within AWS security boundaries.

## Final Fix

Added an inbound rule to bedrock-privatelink-sg allowing TCP 443 from the SG itself, allowing proper interservice communication between my private subnet-bound EC2 instance with Systems Manager. 

## Result

Instance successfully appeared in Systems Manager 

---

## LLM Collaboration Reflection

Initially, the LLM focused on IAM trust relationships, SSM agent state, and endpoint availability. Once I introduced a linked AWS doc about instance visibility, it pivoted to emphasize network-level configuration.

This shift in focus — prompted by human-curated documentation — was a turning point. The combination of LLM-driven diagnostic depth and human-led contextual shifts led to identifying a subtle, low-level misconfiguration that could have been missed in a pure checklist approach.


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

   - LLM-driven checklists are valuable but insufficient without human-led architectural context injection.

   - AI can accelerate diagnostics, but nuanced troubleshooting requires human validation and iterative prompt refinement.
    
   - Periodic reframing / consolidation of progress helps prevent compounding technical debt but requires a keen eye in recognizing when the model is confidently fixated on an erroneous path in the total problem solution space. 

