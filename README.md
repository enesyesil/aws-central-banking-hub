# aws-central-banking-hub


## High-Availability AWS Solution for Centralized Identity in Banks or Umbrella companies managing newly acquired entities.

## Overview

A company that acquires banks needs a way to connect each bank’s AWS account to a central VPC. This allows employees from different banks to log into a central identity system and access shared resources. However, the banks must remain isolated from each other for security and compliance.

To achieve this, Here is the list of Requirements:

### Identity & Access Management (IAM) Requirements

- Centralized Authentication: All bank employees must log into a single, central identity solution
- Role-Based Access Control: Employees should only have access to their bank's AWS resources
- Multi-Factor Authentication (MFA): The system must enforce MFA for authentication to meet security and compliance standards.
- Support for External Identity Providers (Optional): If banks already use Active Directory (AD) or third-party identity providers (Okta, Azure AD), the system should support integration.
- Centralized User Management: Administrators should be able to manage users centrally across all AWS accounts under the umbrella organization.

### Network Connectivity Requirements

- Connect Each Bank’s AWS Network to a Central VPC: Each bank’s AWS VPC must be securely connected to a central VPC that hosts the identity solution.
- Prevent Inter-Bank Communication: Compliance regulations prohibit one bank from accessing another bank’s AWS account. Thus, each bank should be fully isolated from other banks at the network level.
- Use AWS Native Networking Services: The solution must use AWS services like AWS Transit Gateway, AWS PrivateLink, or VPC Peering.

### Security & Compliance Requirements

- Zero Trust Security Model: Each bank must be restricted from accessing other banks’ accounts and must only communicate with the central identity solution.
- Logging & Monitoring: The system must log authentication attempts, failed logins, and any unauthorized access attempts.
- Encryption & Data Protection: All data in transit and at rest must be encrypted using AWS Key Management Service and TLS.
- Compliance with Banking Security Standards: The solution must align with PCI DSS, SOC 2, GDPR, or other banking security requirements.

## High Availability & Scalability Requirements

- Multi-AZ Deployment: The identity solution and network architecture must be deployed across multiple AWS Availability Zones (AZs) to prevent downtime.
- Multi-Region Redundancy (Optional for Disaster Recovery): If needed, the architecture should support failover to a secondary AWS region for disaster recovery.
- Scalability for Future Acquisitions: The system should have an automated onboarding to allow newly acquired banks to be onboarded easily without redesigning the architecture.

### AWS Organization & Account Structure Requirements

- Centralized AWS Account Management: Each bank's AWS account should be managed under a single AWS Organization controlled by the umbrella company.
- Service Control Policies (SCPs) for Security: AWS Organizations SCPs should enforce access restrictions
- Centralized Billing & Governance: The umbrella company should control billing, cost tracking, and security compliance centrally for all bank accounts.

## Table of Contents
1. [Introduction](#introduction)  
2. [Centralized Identity and Access Management (AWS SSO)](#centralized-identity-and-access-management-aws-sso)  
3. [Network Architecture (AWS Transit Gateway)](#network-architecture-aws-transit-gateway)  
4. [Traffic Inspection with AWS Network Firewall](#traffic-inspection-with-aws-network-firewall)  
5. [High Availability and Multi-AZ Design](#high-availability-and-multi-az-design)  
6. [Multi-Region Disaster Recovery (Optional)](#multi-region-disaster-recovery-optional)  
7. [Security and Compliance Enforcement](#security-and-compliance-enforcement)  
8. [AWS Organizations and Service Control Policies (SCPs)](#aws-organizations-and-service-control-policies-scps)  
9. [Automated Onboarding with Terraform](#automated-onboarding-with-terraform)  
10. [Best Practices and Additional Notes](#best-practices-and-additional-notes)
11. [Optional: GitHub Enterprise Self-Hosted on AWS & GitOps Pipeline](#optional-github-enterprise-self-hosted-on-aws--gitops-pipeline)
12. [Choosing Between Third-Party Git Services and Self-Hosted GitOps for Banks](#choosing-between-third-party-git-services-and-self-hosted-gitops-for-banks)
13. [Conclusion](#conclusion)  
14. [References](#references)



## Introduction
This document describes a **multi-account architecture** in AWS designed to host multiple banks (or tenants) securely. It explains how to:

- Use **AWS Transit Gateway (TGW)** as a **hub-and-spoke** for network connectivity.  
- Enforce strict isolation (no direct traffic) between bank accounts.  
- Centralize security, compliance, and identity management.  
- Achieve high availability across multiple Availability Zones (AZs), and optionally multiple regions.  
- Automate onboarding of new bank accounts with **Terraform**.  

The goal is to meet **bank-grade security and compliance** requirements while making management straightforward and scalable.

# 2. Centralized Identity and Access Management (AWS SSO)

- **AWS IAM Identity Center (AWS SSO)** is used for centralized authentication.  
- Each bank’s employees see **only** their own AWS account(s) upon login.  
- **Permission sets** define roles (Admin, ReadOnly, etc.) assigned to bank-specific groups.  
- **MFA (multi-factor authentication)** is strongly recommended.  
- **Short session durations** (e.g., 1 hour for production) increase security.  
- All access is **centrally auditable** (no local IAM users).




**AWS IAM Identity Center (AWS SSO)** focuses on:
- **Centralized login** for multiple AWS accounts.
- Creating **permission sets** that set the user groups (e.g., Admin, ReadOnly) to accounts.
- **Session policies** (MFA, session duration, etc.).
- Role-based access within each account.

**AWS Control Tower**, on the other hand:
- Automates the **provisioning of new AWS accounts** and sets up an initial multi-account structure (e.g., logging account, security account).
- Enforces **blueprinted guardrails** (with Service Control Policies, Config rules, etc.).
- Can automatically **integrate IAM Identity Center** to enable single sign-on in the new accounts.
- Provides a standardized baseline for organizations.

**Key Comparison Points**  
1. **Scope**: 
   - **IAM Identity Center** solves the authentication and authorization problem across accounts (who logs in, with what roles).  
   - **Control Tower** is broader, creating the multi-account environment, enforcing guardrails, and orchestrating best practices.

2. **Use Cases**: 
   - **Use AWS SSO** alone if you already have your multi-account environment set up and only need a unified way to manage users and roles.  
   - **Use Control Tower** if you want to **start fresh** with a guided setup for multi-account governance, including SSO configuration, logging, and security baselines.

3. **Governance vs. Access**: 
   - **Control Tower** gives you an opinionated governance framework (e.g., OUs, SCPs, logging).  
   - **AWS SSO** specifically addresses how users authenticate and access roles in each account.

4. **Integration**: 
   - Control Tower **automates** setting up SSO if desired.  
   - You can also manually configure IAM Identity Center in a pre-existing environment **without** Control Tower.

In summary, **AWS Identity Center** (SSO) manages the identity and access layer, while **AWS Control Tower** can automate setting up and governing multi-account AWS environments (with SSO included). They often complement each other but **serve different purposes** in the overall multi-account strategy. 

If your environment is already **deeply customized or you require special guardrails** not supported by Control Tower, the manual AWS SSO + Organizations route may be more flexible—but also more labor-intensive and prone to misconfigurations. 


## 3. Network Architecture 

**Transit Gateway (TGW) – Hub-and-Spoke Model**  
- **Design**: Each bank’s VPC is a spoke connected to the TGW in a central “shared services” VPC/account.  
- **Isolation**: You can control routing so each bank’s VPC only communicates with specified shared resources, preventing inter-bank traffic.  
- **Scalability**: Supports many VPC attachments (up to thousands), with centralized route table management.  
- **Availability**: Multi-AZ redundancy is built in. If one AZ goes down, TGW routes through another.  
- **Use Case**: Ideal when you need broader Layer 3 IP connectivity between spoke VPCs and a central hub (or on-premises) with full control over routes and security inspection (e.g., Network Firewall).

**AWS PrivateLink**  
- **Design**: Exposes specific services (e.g., an application running behind an NLB) from a “central VPC” to each bank’s VPC.  
- **Isolation**: Each bank’s VPC only “sees” that private endpoint for the one service; no Layer 3 connectivity to the rest of the central VPC.  
- **Granularity**: Allows very strict, service-level access — the bank can only call that one endpoint, not the entire VPC CIDR.  
- **Simplicity**: Reduces the need for managing multiple route tables; just set up VPC endpoints for each service.  
- **Use Case**: Best when banks only need limited, API-like access to a shared service and do not require full network-level routing.

**Key Decision Factors**  
1. **Required Connectivity**  
   - **Transit Gateway**: Use when you need IP-based routing to multiple subnets or services in the central VPC, or if banks require more than a single application endpoint.  
   - **PrivateLink**: Use when banks only need access to a specific service (e.g., a load balancer or API). 

2. **Security & Isolation**  
   - **Transit Gateway**: Offers robust route table isolation (spoke-to-spoke blocking, central inspection) but grants broader network visibility.  
   - **PrivateLink**: Highly restrictive – a bank can only communicate with the exposed endpoint, no broader network traffic.

3. **Complexity & Management**  
   - **Transit Gateway**: Requires planning CIDRs, route tables, attachments, potentially more overhead if you only want to expose a single service.  
   - **PrivateLink**: Easy to set up for a single service; fewer route changes, but can become cumbersome if you have many different services to share (one endpoint per service).

4. **Scalability**  
   - **Transit Gateway**: Scales to thousands of attachments, centralized control.  
   - **PrivateLink**: Scales well for “service endpoints,” but each service typically requires its own endpoint configuration.

In short, **Transit Gateway** is best for broader, network-level connectivity among multiple VPCs, while **AWS PrivateLink** offers more service-specific access with minimal exposure. Depends on the details, We can combine both — using Transit Gateway for general connectivity and PrivateLink for specific private services. 


## 4. Traffic Inspection with AWS Network Firewall
- A **central AWS Network Firewall** in an **inspection VPC** inspects traffic between each bank’s VPC and the hub.  
- The firewall is **stateful**, supports **deep packet inspection**, and can enforce custom rules.  
- Deployed in **multiple AZs** for high availability.  
- **Appliance Mode** in the TGW ensures both outbound and return traffic go through the firewall for stateful inspection.

**Key Points:**
- Blocks malicious or non-compliant traffic.  
- Central rule management; no need for multiple firewall appliances.  
- Defense against lateral movement (e.g., from a compromised VPC).


## 5. High Availability and Multi-AZ Design
- **AWS Transit Gateway** is a managed HA service across AZs.  
- Each VPC attachment uses **subnets in at least two AZs**.  
- **Network Firewall endpoints** are also in multiple AZs.  
- Redundant connections to on-prem or the internet (e.g., VPN or Direct Connect) ensure resilience.  
- A single TGW is typically sufficient for **regional HA**.


## 6. Multi-Region Disaster Recovery (Optional)
To handle region-wide outages:
- Deploy a **similar hub-and-spoke** in a **secondary region**.  
- **AWS SSO** is global; same identity management across regions.  
- **Active/Passive** or **Active/Active** DR approaches:
  - **Active/Passive** uses DNS failover.  
  - **Active/Active** might use **Transit Gateway inter-region peering**.  
- Replicate security controls (Network Firewall, multi-AZ) in the secondary region.  
- Enable **GuardDuty**, **Config**, and **Security Hub** in **both** regions.


## 7. Security and Compliance Enforcement
Several AWS security services are used **org-wide**:

1. **AWS Config**  
   - Records configuration changes and checks them against compliance rules.  
   - Central aggregator in a security account for all bank accounts.

2. **AWS Security Hub**  
   - Aggregates findings from GuardDuty, Inspector, and others.  
   - Runs CIS AWS Foundations checks and best practice audits.  
   - Central delegated admin sees all findings.

3. **Amazon GuardDuty**  
   - ML-based threat detection (suspicious activity, botnet communications, etc.).  
   - Auto-enrolled for new accounts via AWS Organizations.  
   - Central aggregator collects all findings.

4. **AWS Network Firewall & VPC Flow Logs**  
   - Inspects/blocks unwanted traffic.  
   - **Flow Logs** capture IP metadata for further analysis.

5. **AWS CloudTrail**  
   - **Organization trail** logs API calls across all accounts in a central S3 bucket.  
   - Essential for audits.  
   - Optionally enable **CloudTrail Insights** for anomaly detection.

6. **AWS Systems Manager & Amazon Inspector** (optional)  
   - **Patch Manager** auto-patches instances.  
   - **Inspector** scans for vulnerabilities and integrates with Security Hub.

All these tools provide **continuous compliance** for frameworks like PCI-DSS, SOC2, etc.



