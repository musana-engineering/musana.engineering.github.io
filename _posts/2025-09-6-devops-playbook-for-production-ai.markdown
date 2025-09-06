---
layout: post
title: The DevOps Playbook for Production Ready AI – Part 1
date: 2025-09-6 13:32:20 +0300
description: A practical guide to building infrastructure and automation for predictive and generative AI projects on Azure.
img: ml_cover.jpg # Add image post (optional)
fig-caption: # Add figcaption (optional)
tags: [ai, machine-learning, predictive-ai, generative-ai, mlops, devops, kubernetes, azure, azure-machine-learning, cloud-infrastructure, model-deployment, platform-engineering]
---

Many of us have hundreds of photos sitting in our phone galleries that never see the light of day. Snapping a picture is effortless, but capturing one good enough to share takes more effort. The same is true for AI projects. Building a prototype can be quick and exciting, but turning that prototype into something reliable, scalable, and secure in production requires a very different kind of effort.

**You can try to vibe code your way into production but it never ends well**

Running AI in production whether generative or predictive, demands a serving platform that supports continuous deployment, resilient security, and the ability to scale seamlessly as workloads grow. This is not trivial work.

This is where DevOps and Platform Engineering brings value. By applying battle tested practices from modern software delivery such as automation, CI/CD pipelines, monitoring, and infrastructure-as-code (IAC), DevOps/Platform engineering teams are uniquely equipped to help organizations move from endless experimentation to production AI deployments that generate measurable ROI.

In this multi-part blog series, I’ll show you what that effort looks like with a practical, end-to-end implementation for a Predictive AI project on Microsoft Azure.

### Table of Contents
- [Prerequisites](#prerequisites)
- [Introduction](#introduction)
- [Infrastructure Setup ](#infrastructure-setup)
- [Data Acquistion ](#data-acquistion)
   - [Data Connection](#data-connection)
   - [Data Import](#data-import)
- [Putting it all together](#putting-it-all-together)
- [Summary ](#summary)

### Prerequisites
This is a complex technical implementation so before we dive in I assume you have a strong understanding and hands-on experience with the following
- **[Machine Learning](https://mitsloan.mit.edu/ideas-made-to-matter/machine-learning-explained)** concepts including model training, evaluation, and deployment. 
- **[DevOps](https://platformengineering.org/blog/what-is-platform-engineering)** concepts like CI/CD pipelines and version control
- **[Platform Engineering](https://platformengineering.org/blog/what-is-platform-engineering)** concepts and principles
- **[Python](https://learn.microsoft.com/en-us/python/api/overview/azure/ml/?view=azure-ml-py)** programming and the **[Azure ML SDK for Python](https://learn.microsoft.com/en-us/python/api/overview/azure/ml/?view=azure-ml-py)**
- **[Snowflake](https://www.snowflake.com/en/)** data platform.
- **[Azure Kubernetes](https://learn.microsoft.com/en-us/azure/aks/what-is-aks)**
- Infrastructure as code using **[Terraform](https://www.terraform.io/)**

Ensure that **[Argo Events](https://argoproj.github.io/argo-events/)** and **[Argo Workflows](https://argoproj.github.io/workflows/)** are installed on your Kubernetes cluster. Argo Events will handle event-driven triggers for the pipelines, while Argo Workflows will be used to author and execute them

### Introduction

**GloboRealty** is a fictitious real-estate company specializing in residential properties across urban, suburban, rural, and waterfront markets. Over the years, they’ve collected extensive sales records along with property details such as square footage, bedrooms, bathrooms, year built, and condition ratings. They see predictive AI as a way to give buyers, sellers, and investors instant, data-driven insights into property values.

To bring this vision to life, GloboRealty is building on Azure Machine Learning (AML) with strong DevOps and MLOps practices. Their workflow starts with data stored in Snowflake, securely connected to their AML workspace. Pipelines ingest this data into Azure Blob Storage as raw datasets, which are then preprocessed into curated, training-ready datasets. From there, models are trained, evaluated, and deployed through automated CI/CD pipelines ensuring every version is traceable, tested, and monitored in production.

In Part 1, we’ll focus on two key steps:

- **Infrastructure Setup:** Provision the Azure Machine Learning (AML) workspace and configure supporting services such as Blob Storage, Key Vault, and Managed Identity.
- **Data Acquisition:** Establish a secure connection between the AML workspace and Snowflake and Ingest historical property data into Azure Blob Storage as a raw (bronze) dataset, ready for preprocessing in Part 2.

### Infrastructure Setup

Before we can begin acquiring and preparing the raw data, we need a secure and reliable infrastructure foundation in Azure. At the center of the setup is the Azure Machine Learning (AML) workspace. This is where data scientists, engineers, and architects collaborate. It provides experiment tracking, dataset management, and integration with pipelines for automation

![Infra](https://sacoreinfrastate.blob.core.windows.net/assets/aml_infra.jpeg)

Alongside the workspace, several supporting Azure services are provisioned:

- **Azure Storage Account:** For storing datasets at different stages, separating raw ingested data from cleaned, feature engineered datasets.
- **Azure Key Vault:** For storing secrets, connection strings and other sensitive information
- **Managed Identity:** For secure, role-based access to resources like Storage and Key Vault without embedding credentials.

To make the setup repeatable and version controlled, we'll use Terraform to define and provision the infrastructure. 

### Data Acquistion
![Data Pipeline](https://sacoreinfrastate.blob.core.windows.net/assets/data_pipeline_1.jpeg)

### Summary
