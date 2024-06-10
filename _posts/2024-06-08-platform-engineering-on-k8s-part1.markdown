---
layout: post
title: Platform Engineering on Kubernetes with Terraform, Argo, and FastAPI - Part 1
date: 2024-06-8 13:32:20 +0300
description: Explore the practical implementation of Platform Engineering using powerful tools like Terraform, Argo Events, Argo Workflows
img: i-rest.jpg # Add image post (optional)
fig-caption: # Add figcaption (optional)
tags: [platformengineering, kubernetes]
---
The DevOps landscape is constantly evolving, and producing new concepts in its pursuit of automation, collaboration, and efficiency. While some of these concepts are often a repackaging of existing practices, many of them bring meaningful improvements. **[Platform Engineering](https://platformengineering.org/blog/what-is-platform-engineering)** is one of them.

In practical terms, Platform Engineering focuses on designing and building toolchains and workflows that enable self-service capabilities for software engineering teams. As a Platform engineer, you will be required to provide an integrated product most often referred to as an **[Internal Developer Platform](https://internaldeveloperplatform.org/)** covering the operational necessities of the entire lifecycle of an application.

There are many off-the-shelf solutions available for internal developer platforms, but many of them come with significant complexities in setup, maintenance and a bloat of features that may not be relevant or necessary for your specific engineering teams. In many cases, building a custom internal developer platform tailored to your organization's unique needs and requirements is a more reasonable and pragmatic approach.

In this multi-part series, we'll dive deep into a practical implementation of Platform Engineering by building an internal developer platform (IDP) using common tools that platform and DevOps engineers are already familiar with, such as Terraform, Argo Events, Argo Workflows, and FastAPI.

## Table of Contents
- [Introduction ](#introduction)
  - [Platform Capabilities](#platform-capabilities)
  - [Platform Tools ](#platform-tools)
  - [Prerequisites ](#prerequisites)
- [Implementation ](#implementation)
  - [Foundation Architecture ](#foundation-architecture)
  - [Core network ](#core-network)
  - [Core Kubernetes ](#core-kubernetes)
  - [Core Tools ](#core-tools)
  - [Certificate Management ](#certificate-management)
  - [Secret Management ](#secret-management)
  - [Ingress Management ](#ingress-management)
- [Summary ](#summary)

## Introduction
**A Unified Developer Experience:** Imagine a centralized platform where developers can call an api to provision and manage infrastructure, automate workflows, and build and deploy applications with ease. This platform would serve as a one-stop shop, eliminating the need for disparate tools and manual processes, ultimately reducing complexity and increasing productivity. To better illustrate what we will be building throughout this multi-part series, let's take a look at the following architectural diagram

![main](https://github.com/musana-engineering/idp/assets/151420844/e164cc3b-c7e9-4289-a9fc-a85d41369da1)

- **1)** Developers interact with the IDP through the APIs exposed by FastAPI. These APIs serve as the entry point for developers to trigger various actions, such as provisioning infrastructure, deploying applications, or executing workflows.
- **2)** When a developer interacts with the FastAPI endpoints, it generates an event that is consumed by Argo Events webhook sensors.
- **3)** Based on the events received from FastAPI (via Argo Events), Argo Workflows is triggered to orchestrate and execute the desired actions or workflows.

## Platform Capabilities
Our internal developer platform will be built to include 5 core capabilities.

- **Infrastructure provisioning:** Enable developers to create cloud infrastructure resources that adhere to security and performance best practices while abstracting complexities such  as networking and security.  

- **Environment Deployment:** Enable developers to create new and fully provisioned environments whenever needed and also delete them when nolonger needed.

- **Application Deployment:** Enable developers to deploy applications based on various events, such as code commits or manual triggers,

- **Application Configuration:** Enable developers configure applications based on various events, such as code commits or manual triggers

- **Access Control:** Manage who can do what in a scalable way.

## Tools
Before diving into the implementation details, let's familiarize ourselves with the key tools and technologies that will power our internal developer platform.

- **[Kubernetes](https://kubernetes.io/)**, an open source system for automating deployment, scaling, and management of containerized applications will be the underlying foundation upon which our internal developer platform will be built. All the components of our solution, including Terraform, Argo Events, Argo Workflows, and FastAPI will be running on Kubernetes

- **[Terraform](https://www.terraform.io/)**, an infrastructure as code tool will be used to define and manage all cloud infrastructure resources created through our platform. This process will be abstracted from the developer and occur seamlessly behind the scenes when an API endpoint is called, such as when creating a dev environment

- **[Argo Events](https://argoproj.github.io/argo-events/)**, an event-driven workflow automation framework for Kubernetes, will enable us to build event-driven applications and workflows. It seamlessly integrates with various event sources, such as webhooks, message queues, and Kubernetes resources, allowing us to trigger actions and workflows based on specific events. In our implementation, Argo Events will react to events generated by developers interacting with the FastAPI endpoints, enabling automated workflows and orchestration.

- **[Argo Workflows](https://argoproj.github.io/workflows/)**, a container-native workflow engine, will be our CI/CD pipelining solution.This includes cloud resource provisioning, configuration, and application deployments. Using Argo Workflow templates, we will create Terraform tasks for provisioning and Ansible scripts for configuration.

- **[FastAPI](https://fastapi.tiangolo.com/)**, a modern high-performance web framework for building APIs with Python, will be the frontend of our internal developer platform. Developers will interact with the platform's capabilities through intuitive APIs exposed by FastAPI. Additionally, FastAPI's built-in Swagger UI will provide a user-friendly interface for exploring and testing the available APIs, fostering self-service. 

## Prerequisites
To follow along with this multi-part series and implement the solution described above, we'll be using Microsoft Azure and Azure Kubernetes Service (AKS). To get started, you'll need the following prerequisites:

- **[A Microsoft Azure](https://azure.microsoft.com/en-us/pricing/offers/ms-azr-0044p)** account with an active subscription and necessary permissions to create and manage resources.
- **[Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)**, a command-line tool for interacting with your Kubernetes cluster.
- **[jq](https://jqlang.github.io/jq/)**, a lightweight and flexible command-line JSON processor.
- **[Helm](https://jqlang.github.io/jq/)**, a package manager for Kubernetes that simplifies the process of managing Kubernetes applications.
- **[Visual Studio Code](https://code.visualstudio.com/)** to make it easier to work with the code samples and configuration files provided throughout the series.

**Note:** Having a good understanding of Kubernetes concepts and architecture will be beneficial as we'll be deploying and interacting with Kubernetes throughout the series. Please note the principles and tools discussed in this series are transferable to other cloud providers and Kubernetes distributions, allowing you to adapt the solution to your preferred environment.

## Implementation: 
In this first part of the series, we’ll focus on setting up the foundational infrastructure and services required for our internal developer platform implementation in Microsoft Azure. We will define our infrastructure requirements using Terraform. This includes provisioning a Virtual Network, Azure Kubernetes cluster, configuring network security settings and setting up the necessary storage resources.

- ### Foundation Architecture
![idp-network](https://github.com/musana-engineering/idp/assets/151420844/e448acb6-7001-4cba-a0a6-59ccb9af21c2)
- ### Core network
Follow the steps below to deploy the core network.

Create an **[Azure service principal](https://learn.microsoft.com/en-us/cli/azure/azure-cli-sp-tutorial-1?tabs=bash)** to be used for Terraform provider authentication

{% highlight javascript %}
// Login to Azure CLI and set the subscription to use
az login
az account set -s "your_subscription_id_here"

// Set the following Environment Variables
export ARM_CLIENT_ID="your_client_id_here"
export ARM_CLIENT_SECRET="your_client_secret_here"
export ARM_TENANT_ID="your_tenant_id_here"

// Clone the project repository
git clone https://github.com/musana-engineering/idp.git

// Navigate to the network directory
cd idp/core/network

// Generate and review the Terraform plan
terraform init && terraform plan

// Provision the infrastructure.
terraform apply
{% endhighlight %}

- ### Core Kubernetes
Follow the steps below to deploy the Core Kubernetes cluster.

{% highlight javascript %}
// Navigate to the aks directory
cd idp/core/aks

// Generate and review the Terraform plan
terraform init && terraform plan

// Provision the infrastructure.
terraform apply
{% endhighlight %}

- ### Core Tools
![platform_tools](https://github.com/musana-engineering/idp/assets/151420844/81dac169-b1b7-4ec7-80c0-a23b12962bb1)
 - ### Certificate Management
Securing our applications with SSL/TLS certificates is a critical aspect of our Platform. We will use **[Cert Manager](https://cert-manager.io/)** to facilitate the issuance, renewal, and management of SSL/TLS certificates from **[Letsencrypt](https://letsencrypt.org/)** and securely store them as a Kubernetes secrets within the Cert-Manager namespace. To meet the diverse needs of our platform, we require the certificates to be accessible across multiple namespaces. To achieve this, we will leverage the  **[Kubernetes Replicator](https://github.com/mittwald/kubernetes-replicator)**, enabling seamless replication of secrets throughout our Kubernetes environment.

{% highlight javascript %}
For the examples in this series, I am using the packetdance.com DNS zone and requesting a wildcard certificate *.packetdance.com from Letsencrypt. This wildcard certificate will allow us to secure all subdomains under packetdance.com with a single SSL/TLS certificate, simplifying the management process.

You MUST replace packetdance.com with a domain that you own and have control over. Using a domain you don't own or control can lead to security issues and potential certificate issuance failures.
{% endhighlight %}

 - ### Secret Management
Safeguarding sensitive data is crucial for our platform's security. We will use Azure Key Vault as our centralized repository for securely storing all secrets, leveraging industry-standard encryption and access control. To seamlessly integrate Azure Key Vault with our Kubernetes clusters, we'll utilize the **[Exetrnal Secrets Operator](https://external-secrets.io/latest/)** to bridge our Kubernetes cluster and Azure Key Vault, allowing us to securely retrieve secrets directly into the relevant namespaces without compromising security or increasing complexity.

 - ### Ingress Management
Some of the core components like Argo Workflows, ArgoCD, Argo Rollouts, and other applications hosted on our platform provide web interfaces and dashboards that will need to be accessed by users outside the Kubernetes cluster. To make these available, we will implement the Nginx Ingress Controller to act as a reverse proxy and load balancer, routing incoming traffic to the appropriate services based on the requested URL. We will configure Nginx Ingress to handle host-based routing, allowing us to access different web UIs for different purposes. For example, we will  set up the following ingress endpoints:

- **argoworkflows.packetdance.com** to access the Argo Workflows UI
- **argocd.packetdance.com** to access the ArgoCD UI
- **argorollouts.packetdance.com**  to access the Argo Rollouts UI
- **api.packetdance.com** to access our FastAPI application

Follow the steps below to deploy the platform tools.

{% highlight javascript %}
// Navigate to the aks directory
cd idp/core/tools

// Generate and review the Terraform plan
terraform init && terraform plan

// Provision the infrastructure.
terraform apply
{% endhighlight %}

At this stage, we have deployed the foundational infrastructure, tools, and services for our internal developer platform. You can now check your Azure subscription to review all the resources created so far.

{% highlight javascript %}
// List all resource groups 
az group list -o table

Name              Location    Status
----------------  ----------  ---------
RG-core           westus      Succeeded
RG-idp-net        westus3     Succeeded
RG-idp-aks        westus3     Succeeded
RG-idp-aks-nodes  westus3     Succeeded
{% endhighlight %}

Additionally, connect to your Kubernetes cluster to verify that the platform tools have been successfully installed

{% highlight javascript %}
// List all deployments in all namespaces
kubectl get deployments --all-namespaces=true
argo
argo-events
argo-rollouts
argocd
cert-manager
external-dns
external-secrets
{% endhighlight %}

### Summary
In this first part of the series, we laid the foundation for building an internal developer platform on Kubernetes. We defined the core components and tools that will power our platform, including Kubernetes as the underlying infrastructure, Terraform for provisioning cloud resources, Argo Events for event-driven automation, Argo Workflows for CI/CD pipelines, and FastAPI as the frontend API layer.

We started by provisioning the core network infrastructure using Terraform, including a virtual network and an Azure Kubernetes cluster. We then deployed essential platform tools like Cert-Manager for SSL/TLS certificate management, External Secrets Operator for secure secret management, and Nginx Ingress for routing incoming traffic.

Throughout the implementation process, we emphasized the importance of security, scalability, and developer experience. By abstracting complexities and providing self-service capabilities, our internal developer platform aims to empower developers, streamline workflows, and foster a more efficient and collaborative software engineering culture.

In the upcoming parts of this series, we will dive deeper into the implementation details, covering the integration of Argo Events, Argo Workflows, and FastAPI to enable event-driven automation, CI/CD pipelines, and intuitive API interfaces for developers to interact with the platform.