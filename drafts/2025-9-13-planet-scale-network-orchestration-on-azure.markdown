---
layout: post
title: Planet Scale Virtual Network Orchestration on Azure
date: 2025-09-12 13:32:20 +0300
description: A practical guide to designing, automating, and orchestrating global networks for highly available applications on Microsoft Azure
img: main1.jpeg # Add image post (optional)
fig-caption: # Add figcaption (optional)
tags: [ai, mlops, devops, kubernetes, azureml, snowflake, argo]
---

Microsoft Azure’s global network is immense: more than 60 regions, 220+ datacenters, 170+ edge sites, and 165,000 miles of fiber. For platform engineers, this is a unique canvas for building planet-scale architectures that power globally available applications.

The challenge? Orchestration at scale. Manually wiring regions, enforcing security policies worldwide, and ensuring resilience is neither practical nor sustainable.

The solution lies in automation and intent-based management. At the core of this approach is Azure Virtual Network Manager (AVNM), which provides centralized connectivity management and enforces security at scale.

This guide demonstrates how to use AVNM and Infrastructure as Code (IaC) to design, automate, and operate a secure, resilient, and scalable global network illustrated through the lens of a fictitious company expanding its global footprint.

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

**GloboJava** is a fictional company known for its coffee-based products—specialty beverages and pastries. Its mission is to deliver an exceptional coffee experience, reliably and at speed, wherever customers place an order.

Currently operating across the Americas, GloboJava is expanding into Europe, Africa, Oceania, and Southeast Asia. To enable this global presence, the Platform Engineering team must design and manage a secure, scalable, and resilient network architecture to support millions of daily transactions.

The team faces four key requirements:

- **Low Latency Everywhere:** Customer app experience should be the same regardless of location.
- **Uncompromising Security:** A zero-trust network with strict segmentation and centralized inspection of all cross-region traffic.
- **Operational Resilience:** A single region failure must not disrupt operations across other regions and continents.
- **Velocity and Scale*:** New regions must be provisioned automatically, with infrastructure defined as code.

### Architecture
<img src="../assets/img/network_architecture.jpeg"/>

The initial assessment confirmed that manually peering dozens of VNets across regions would be unmanageable and error prone. Instead, the team designed an orchestrated, intent-driven architecture leveraging Azure Virtual Network Manager (AVNM)

At a high level, the network design includes regional hubs as follows:

- **Americas:** hub-eastus, hub-westus
- **Europe:** hub-northeurope, hub-westeurope
- **Asia Pacific:** hub-southeastasia, hub-australiaeast
- **Africa:** hub-southafricanorth

Each hub hosts an Azure Firewall and connects to two spokes (**prod** and **nonprod**). Local traffic remains within a region for lowest latency, while hubs provide secure inter-region routing.

### Orchestration

Managing peering, security, and routing across 8 hubs and 16+ spokes manually would quickly spiral into complexity. The solution: Azure Virtual Network Manager (AVNM) as the global management plane.

- ### Dynamic Group Management: 

AVNM supports dynamic network groups based on tags. For example:

- Tag hubs with **network_role = hub**
- Tag production spokes with **env = prod**
- Tag non-production spokes with **env = nonprod**

Groups then form automatically:

- global-hubs → network_role = hub
- all-prod-spokes → env = prod
- all-nonprod-spokes → env = nonprod

When a new spoke is tagged **env=prod**, AVNM automatically onboards it removing manual intervention and ensuring consistency.

- ### Connectivity Configuration:

AVNM applies a Mesh Connectivity Configuration across the global-hubs group. This establishes VNet peering automatically, ensuring every hub is fully connected without manual setup.

Example traffic flow:
- **spoke-prod-eastus → hub-eastus → hub-westeurope → spoke-prod-westeurope**

- ### Security Admin Configuration

AVNM enforces intent-based security rules globally, creating a protective layer above VM level NSGs. These guardrails prevent misconfigurations, such as accidental traffic between prod and nonprod, ensuring uniform compliance across continents.

- ### Routing and DNS

**Firewall Manager:** A single global Firewall Policy is attached to all regional firewalls. Changes propagate worldwide in minutes, ensuring consistent enforcement.

**Unified DNS:** A global Azure Private DNS Zone (e.g., globojava.com) links to all VNets. Developers can use intuitive service names like database.westeurope.globojava.com, with DNS resolving seamlessly to the correct regional private IP.

### Pipeline Setup

To make the network orchestration repeatable, reliable, and scalable, GloboJava uses Argo Workflows running on AKS to automate deployments. The workflows are designed around GitOps principles: infrastructure changes are version-controlled in Git, and pipelines reconcile desired state with actual Azure resources. (See my Platform Engineering with Kubernetes here **[Part 1](https://musana.engineering/platform-engineering-on-k8s-part1/)** and **[Part 1](https://musana.engineering/platform-engineering-on-k8s-part2/)**)

Two separate pipelines provide the right balance between safety and speed:

**Infrastructure Deployment Pipeline:** This triggers on any push to main and impacts resources such as VNets, AVNM groups, hub-and-spoke definitions, and CIDR changes

{% highlight shell %}

{% endhighlight %}

**Firewall Policy Pipeline:** 

Handles frequent firewall rule updates. This pipeline is designed to be fast-moving since security rules evolve frequently. Global consistency is guaranteed because all firewalls inherit from the same AVNM-managed policy.

### Summary

In the past, managing a network of this scale meant battling complexity and risk. Today, with AVNM and declarative IaC, orchestration is not only possible but practical. For GloboJava, this means focusing on coffee not network plumbing. Their global network now automatically connects, secures, and scales across regions, ensuring customers everywhere experience the same fast, reliable service.

This is the promise of modern cloud networking: where global complexity is tamed by intent-based policies and declarative code.