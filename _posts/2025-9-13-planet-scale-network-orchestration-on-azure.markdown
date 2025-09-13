---
layout: post
title: Planet Scale Network Orchestration on Azure
date: 2025-09-12 13:32:20 +0300
description: A practical guide to designing, automating, and orchestrating global networks for highly available applications on Microsoft Azure
img: main1.jpeg # Add image post (optional)
fig-caption: # Add figcaption (optional)
tags: [ai, mlops, devops, kubernetes, azureml, snowflake, argo]
---

Microsoft Azure’s network is massive with 60+ regions, 220+ datacenters, 170+ edge sites, and 165,000 miles of fiber. For platform engineers, it’s a unique canvas to build planet scale network architectures for globally available applications. The challenge? Orchestration at scale. Manually wiring regions, enforcing security policies worldwide, and ensuring resilience isn’t practical.

The answer is automation and intent based management. At the center of this approach is Azure Virtual Network Manager (AVNM), which unifies regional VNets into a cohesive, intelligent global fabric.

This post shows how to use AVNM and Infrastructure as Code (IaC) to design, automate, and manage a secure, scalable, and resilient global network on Azure through the lense of a fictious company looking to expand and establish it's presence globally.

### Table of Contents
- [Introduction](#prerequisites)
- [GloboJava](#introduction)
- [Architecture ](#infrastructure-setup)
- [Orchestration](#data-acquistion)
   - [Connectivity Configuration](#data-connection)
   - [Security Configuration](#data-import)
   - [Routing Configuration](#data-asset)
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

Currently, GloboJava operates across the Americas, but the company is preparing to expand its footprint to Europe, Africa, Oceania, and Southeast Asia. To support this global presence, the Platform Engineering team has been tasked with designing, implementing, and managing a network capable of operating within the required performance envelope to meet the demands of a worldwide customer base.

### Architecture
<img src="../assets/img/network_architecture.jpeg"/>

Global Footprint:
- Americas: East US (Hub) & West US (Hub)
- Europe: North Europe (Hub) & West Europe (Hub)
- Asia Pacific: Australia East (Hub) & Southeast Asia (Hub)
- Design Rules:

Each region has its own Hub VNet. This is the key change.

Each Hub contains an Azure Firewall Premium (for intra-region traffic inspection).

Each Hub has two Spoke VNets (e.g., in East US: prod-eastus and nonprod-eastus).

Spokes peer directly to their regional Hub. All traffic to/from a spoke is forced through its local Hub's firewall (e.g., prod-eastus -> hub-eastus -> Internet).

All Hubs are interconnected in a global mesh via Azure Virtual Network Manager (AVNM). This allows a spoke in East US to reach a spoke in Southeast Asia via the path: spoke-eastus -> hub-eastus --(AVNM Mesh)--> hub-southeastasia -> spoke-southeastasia.

Spokes in the same region can talk directly through the regional hub's firewall, which is low-latency.