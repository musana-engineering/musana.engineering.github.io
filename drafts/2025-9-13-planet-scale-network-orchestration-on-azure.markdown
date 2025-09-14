---
layout: post
title: Planet Scale Virtual Network Orchestration on Azure
date: 2025-09-12 13:32:20 +0300
description: A practical guide to designing, automating, and orchestrating global networks for highly available applications on Microsoft Azure
img: main1.jpeg # Add image post (optional)
fig-caption: # Add figcaption (optional)
tags: [ai, mlops, devops, kubernetes, azureml, snowflake, argo]
---

Microsoft Azure’s network is massive with 60+ regions, 220+ datacenters, 170+ edge sites, and 165,000 miles of fiber. For platform engineers, it’s a unique canvas to build planet scale network architectures for globally available applications. The challenge? Orchestration at scale. Manually wiring regions, enforcing security policies worldwide, and ensuring resilience isn’t practical.

The answer is automation and intent based management. At the center of this approach is Azure Virtual Network Manager (AVNM), which manages virtual network connectivity and enforces security rules at scale.

This post shows how to use AVNM and Infrastructure as Code (IaC) to design, automate, and manage a secure, scalable, and resilient global network on Azure through the lense of a fictious company looking to expand and establish it's presence globally.

### Table of Contents
- [Introduction](#introduction)
- [Architecture ](#architecture)
- [Orchestration](#orchestration)
   - [Dynamic Group Management](#dynamic-group-management)
   - [Connectivity Configuration](#connectivity-configuration)
   - [Security Configuration](#securiy-configuration)
   - [Routing Configuration](#routing-configuration)
- [Pipeline Setup](#pipeline-setup)
- [Summary ](#summary)

### Prerequisites
This guide covers advanced network architecture on Microsoft Azure. Before diving in, you should have a strong understanding and experience with the following;

- **[Azure Networking](https://learn.microsoft.com/en-us/azure/networking/fundamentals/networking-overview):** A solid grasp of fundamental services like Azure Virtual Networks (VNets), VNet Peering, User-Defined Routes (UDRs) and Azure DNS
- **[Azure Network Security ](https://learn.microsoft.com/en-us/azure/security/fundamentals/network-overview):** Hands-on experience with Azure Firewall and Network 
- **[DevOps](https://devops.com/)** concepts like CI/CD pipelines and version control
- **[Platform Engineering](https://platformengineering.org/blog/what-is-platform-engineering)** concepts and principles

### Introduction

**GloboJava** is a fictitious company specializing in coffee based products, including beverages and pastries. Its mission is to deliver exceptional coffee experiences, ensuring fast and reliable service no matter when or where customers place their orders.

Currently, GloboJava operates across the Americas, but the company is preparing to expand its footprint to Europe, Africa, Oceania, and Southeast Asia. To support this global presence, the Platform Engineering team has been tasked with designing, implementing, and managing a highly secure, scalable, reliable, and resilient network architecture to support this global presence.

The team must meet the following requirements

- Low Latency Everywhere: A customer ordering via mobile app in Lisbon must experience the same instantaneous response as a customer in Chicago.
- Uncompromising Security: The network handling millions of daily transactions must be architected for zero-trust, with strict segmentation and centralized inspection for all cross-region traffic.
- Operational Resilience: The failure of a single region must be isolated and must not impact operations on other continents.
- Velocity and Scale: The architecture must enable, not hinder, rapid expansion into new markets. Provisioning a new region must be an automated, code driven process.

### Architecture
<img src="../assets/img/network_architecture.jpeg"/>

The team’s initial assessment concluded that a traditional approach manually peering a sprawling web of virtual networks would be unmanageable, insecure, and unable to meet their scalability and reliability goals. They needed an orchestrated approach.

The following design is their blueprint for a planet scale Azure network, leveraging automation and intent-based policies to turn global complexity into a manageable, seamless platform.

- Americas: hub-eastus, hub-westus
- Europe: hub-northeurope, hub-westeurope
- Asia Pacific: hub-southeastasia, hub-australiaeast
- Africa: hub-southafricanorth

Each hub contains an Azure Firewall and serves two spoke VNets (spoke-prod, spoke-nonprod) in its local region. This ensures traffic between a user and a VM in the same region never leaves the region, providing the lowest possible latency.

### Orchestration

The Orchestration Challenge: Managing peering, security, and routing across 8 hubs and 16+ spokes manually is a recipe for human error and inconsistency. To solve this problem, the platform engineering team levergaed the Azure Virtual Network Manager (AVNM) to transform this complexity into simplicity.

AVNM provides the Centralised Management plane for our entire global network.

- ### Dynamic Group Management: 

Use Azure Tags to dynamically group our resources.

- Tag every hub VNet with **network_role = hub**.
- Tag every production spoke with **env = prod**.
- Tag every non-production spoke with **env = nonprod**.

In AVNM, we create dynamic membership network groups based on these tags.

- global-hubs" Group: (network_role Equals hub)
- all-prod-spokes" Group: (env Equals prod)
- all-nonprod-spokes" Group: (env Equals nonprod)

When a new spoke VNet is created in a new region tomorrow with the tag env=prod, AVNM automatically detects it and adds it to the correct group. The orchestration is dynamic and automatic.

- ### Connectivity Configuration:

We apply a Mesh Connectivity Configuration to the "global-hubs" group. AVNM automatically establishes global VNet peering between every single hub VNet. It manages the peering relationships, ensuring the entire backbone is fully connected without a single manual peering request from us.

Traffic Flow: A VM in spoke-prod-eastus can now talk to a database in spoke-prod-westeurope. The path is: spoke-prod-eastus -> hub-eastus --(AVNM Mesh)--> hub-westeurope -> spoke-prod-westeurope

- ### Security Admin Configuration

We define intent-based security policies that AVNM enforces globally. These rules are enforced at the network level, before traffic even reaches a VM's NSG. They provide a critical layer of compliance, ensuring a developer can never accidentally allow access from nonprod to prod, no matter what they configure on their VM. This guarantees consistent security across the entire globe.

- ### Firewall Manager and Unified DNS

We create a single, global Azure Firewall Policy and associate it with all eight Azure Firewall instances in our regional hubs. Every packet leaving any spoke, destined for the internet or another region, is inspected against the same set of security rules. We get Scale and Consistency in our security posture. A change to the firewall policy is deployed globally within minutes.

We use a central Azure Private DNS Zone for globojava.com and link it to every single VNet (all 8 hubs and 16 spokes). A microservice in spoke-prod-eastus can connect to database.westeurope.globojava.com. The name resolves automatically to the private IP address in the spoke-prod-westeurope VNet, enabling seamless, developer-friendly service discovery across the entire planet.

### Pipeline Setup

To achieve true GitOps for our network, we implement two separate CI/CD pipelines using Argo Workflows. This separation of concerns is critical: infrastructure changes (VNets, peering) are high-impact and require rigorous validation, while firewall rule updates are more frequent and need a faster, more agile process.

**Infrastructure Deployment Pipeline**

This pipeline is triggered for changes to the Terraform code in the environments/ directory (e.g., adding a new region, modifying CIDR blocks). It is designed for stability and thorough planning. The pipeline is triggered for pushes to the main branch in the environments/prod/ path of the Git repository.

**Pipeline Definition**

{% highlight shell %}

{% endhighlight %}

### Summary

Before, managing a network of this scale meant grappling with overwhelming complexity. Today, with Azure's orchestration tools, the paradigm has shifted.

For GloboJava, this means they can focus on brewing the perfect cup of coffee, confident that their global network is not just built, but intelligently orchestrated—automatically connecting, securing, and scaling to serve customers anywhere in the world. By combining the hub-per-region model with AVNM for seamless mesh connectivity, Firewall Manager for centralized policy, and Terraform for automated deployment, the Platform Engineering team has provided the foundation for a truly global empire.

This is the power of modern cloud networking: where complexity is managed not by manual effort, but by declarative code and intelligent orchestration.