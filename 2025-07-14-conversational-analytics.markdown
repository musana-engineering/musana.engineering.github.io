---
layout: post
title: Conversational Analytics on SQL Server with Azure OpenAI and FastAPI
date: 2025-07-14 13:32:20 +0300
description: Enabling natural language queries from a Structured Database
img: ml_cover.jpg 
fig-caption: # Add figcaption (optional)
tags: [analytics, openai, analytics, ai, agentic, text-to-sql, azure-openai]
---
For most organizations, marketing analysts rely on dashboards, spreadsheets, or SQL-savvy colleagues to access insights about campaign performance, customer segmentation, or regional trends. But what if your internal tools could let them simply ask a question in plain English — like “How many new customers signed up last month from California?” — and get an accurate, live answer in seconds?

That’s the goal of conversational analytics: enabling business teams to work directly with raw data using natural language, without writing SQL or waiting on data teams.

In this blog post, I’ll walk you through how to build a production-grade conversational analytics backend using Azure OpenAI, SQL Server, and FastAPI. This solution is designed for engineering teams that want to embed natural language querying into existing internal tools, not build a full user-facing frontend.

- The architecture follows DevOps best practices:
- Version-controlled code in Azure Repos
- Infrastructure as code (IaC) using Terraform
- Secret management using Azure Key Vault
- Containerized backend deployed via Azure Container Apps

### Table of Contents
- [Architecture Overview](#introduction)
- [What is Conversational Analytics?](#what-is-conversational-analytics?)
- [About the Database](#about-the-database)
- [Step-by-Step Implementation](#step-by-step-implementation)
     - [Set Up Azure DevOps Project](#set-up-azure-devOps-project)
     - [Provision Azure Infrastructure](#provision-azure-infrastructure)
     - [Build the API with FastAPI](#build-the-api-with-fastapi)
     - [Dockerize the App ](#dockerize-the-app)
     - [Deploy to Azure Container Apps](#deploy-to-azure-container-apps)
- [Harden for Production](#harden-for-production)
- [Conclusion](#conclusion)

### Prerequisites
Before you get started, please ensure you have the following:

- **[Microsoft Azure account](https://azure.microsoft.com/en-gb/pricing/offers/ms-azr-0044p/):** This is where we will set up the cloud infrastructure needed to support the data ingestion processes.
- **[Azure service principal](https://learn.microsoft.com/en-us/cli/azure/azure-cli-sp-tutorial-1?tabs=bash):** For Terraform provider authentication
- **[Terraform](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli):** You will need this to provision the cloud resources we need to support the data ingestion processes.
- **[Python](https://learn.microsoft.com/en-us/python/api/overview/azure/ml/?view=azure-ml-py)** programming

### Architecture Overview
At a high level, this system allows a user (e.g., analyst) to send a natural language question to a REST endpoint. The backend:

- Uses Azure OpenAI (GPT-4) to generate SQL
- Executes the SQL against a SQL Server database
- Returns the query , query result and a plain english explanation

All of this is exposed via a containerized FastAPI service deployed to Azure Container Apps.

![image](https://github.com/user-attachments/assets/c2c624ed-cb22-48d5-bc38-f8c6487b0f38)


### Conclusion


