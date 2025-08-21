# Security Pillar – AWS Well-Architected Framework


## Introduction
The **Security Pillar** focuses on protecting information, systems, and assets while delivering business value. It highlights implementing strong security foundations, automating protections, and preparing for security incidents.

## Design Principles
- **Implement a strong identity foundation** – Use least privilege and centralized identity.
- **Enable traceability** – Monitor, log, and audit actions.
- **Apply security at all layers** – Defense in depth across network, application, and data.
- **Automate security best practices** – Reduce human error.
- **Protect data in transit and at rest** – Encrypt and control access.
- **Keep people away from data** – Use automation to minimize direct access.
- **Prepare for security events** – Incident response planning and testing.

## Key Best Practices
- Centralize IAM with **MFA and role-based access**.
- Enable **logging and monitoring** (CloudTrail, GuardDuty, Security Hub).
- Apply **network segmentation** (VPC, Security Groups).
- Use **encryption services** (KMS, ACM).
- Regularly **test incident response plans**.

## Goal
Build resilient workloads with confidentiality, integrity, and availability protected by automated, traceable, and well-tested security controls.

Read more: https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/welcome.html
