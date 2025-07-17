---
layout: post
title: Conversational Analytics on SQL Server with Azure OpenAI
date: 2025-07-14 13:32:20 +0300
description: Enabling natural language queries from a Structured Database
img: ml_cover.jpg 
fig-caption: # Add figcaption (optional)
tags: [analytics, openai, analytics, ai, agentic, text-to-sql, azure-openai]
---
For most organizations, marketing analysts rely on dashboards, spreadsheets, or SQL-savvy colleagues to access insights about campaign performance, customer segmentation, or regional trends. But what if your internal tools could let them simply ask a question in plain english like “How many new customers signed up last month from California?” and get an accurate, live answer in seconds?

That’s the goal of conversational analytics: enabling business teams to work directly with raw data using natural language, without writing SQL or waiting on data teams.

In this blog post, I’ll walk you through how to build a production-grade conversational analytics backend using Azure OpenAI, SQL Server, and FastAPI. This solution is designed for engineering teams that want to embed natural language querying into existing internal tools.

- The architecture follows DevOps best practices
- Version-controlled code in GitHub
- Infrastructure as code (IaC) using Terraform
- Secret management using Azure Key Vault
- Containerized backend deployed via Azure Container Apps

### Table of Contents
- [What is Conversational Analytics?](#what-is-conversational-analytics?)
- [Architecture Overview](#introduction)
- [Database Overview](#database-overview)
- [Step-by-Step Implementation](#step-by-step-implementation)
     - [Set Up Git Repository](#set-up-azure-devOps-project)
     - [Provision Azure Infrastructure](#provision-azure-infrastructure)
     - [Build the API with FastAPI](#build-the-api-with-fastapi)
     - [Dockerize the App ](#dockerize-the-app)
     - [Deploy to Azure Container Apps](#deploy-to-azure-container-apps)
- [Harden for Production](#harden-for-production)
- [Conclusion](#conclusion)

### Prerequisites

Before getting started, make sure you have the following tools and access in place:

- **[Microsoft Azure Subscription](https://azure.microsoft.com/en-gb/pricing/offers/ms-azr-0044p/):** In which to provision the cloud infrastructure (SQL Server, OpenAI, Key Vault, ACR, and Container Apps).

- **[Terraform](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli):** For defining and deploying Azure infrastructure as code.

- **[Azure Service Principal](https://learn.microsoft.com/en-us/cli/azure/azure-cli-sp-tutorial-1?tabs=bash):** For authenticating Terraform when provisioning the Azure resources. 

- **[Python](https://www.python.org/):** For building the FastAPI application backend. 

- **[Docker](https://www.docker.com/):** For containerizing the FastAPI app before deploying it to Azure Container Apps.

- **[Azure DevOps Project](https://www.docker.com/):** For version control using Git and CI/CD pipelines support.

### What is Conversational Analytics

In the context of the solution we are going to build, conversational analytics means enabling users (e.g., business analysts, marketing teams) to interact directly with live data in Azure SQL just by typing questions into a simple text box or REST client. Users simply ask questions like:

“How many customers placed orders last month?”
“What was our top selling product in California?”

This bridges the gap between technical data systems and non technical users, removing the need for SQL expertise or waiting on data teams. In short, conversational analytics makes data analysis as simple as having a conversation, while still leveraging the power and structure of enterprise-grade data systems like SQL Server

### Architecture Overview

At a high level, this solution enables business users (e.g. marketing analysts) to query live data using natural language via a secure REST API:

![image](https://rpgcdnfiles.blob.core.windows.net/devops/SqlCopilot.png)

**User Interaction:** 

- Users send natural language questions (e.g., "How many customers placed orders last month?") to the /sqlcopilot endpoint exposed by the FastAPI service.

**Backend Logic**

- The incoming question is passed to an Azure OpenAI powered GPT-4 assistant. 
- The assistant uses a schema-aware system prompt to translate the natural language into a valid SQL query.
- The query is executed against the Azure SQL Server (which hosts the AdventureWorks database). 
- Results are returned, along with: the original SQL query, the structured data result (in JSON) and a plain English explanation of what the query did. The /sqlcopilot endpoint responds with all 3 elements: query, results, and explanation. This empowers users to learn SQL over time while getting fast answers.

### Database Overview

To demonstrate the power of conversational analytics, we'll connect our assistant to the AdventureWorksLT sample database, a widely-used dataset provided by Microsoft to simulate real world retail and sales operations. This Lightweight Sales schema (SalesLT) includes:

- Customer: customer info like name, contact, company
- Address: street, city, state, ZIP
- CustomerAddress: mapping between customers and addresses
- SalesOrderHeader, SalesOrderDetail: orders and their items
- Product: product catalog
- ProductCategory: category hierarchy

This schema is realistic enough to mirror many internal business use cases making it ideal for prototyping AI driven analytics on real looking data without exposing sensitive production systems.

- **Schema Prompting**

One of the core challenges with natural language to SQL is schema grounding. The LLM must understand:

- What tables exist
- How they relate to each other
- Which fields are available
- What to filter on

If the LLM guesses, it hallucinates generating invalid table names or joins. To prevent this, we will use a technique called schema prompting to help the GPT-4 model generate accurate SQL queries. This means we explicitly provide the model with a structured description of the database including table names, column names, and their relationships along with the user’s natural language question.

It looks something like:

{% highlight javascript %}
SCHEMA = """
Tables:
- Customer(CustomerID, FirstName, LastName, EmailAddress)
- Address(AddressID, StateProvince, City, PostalCode)
- CustomerAddress(CustomerID, AddressID)
- SalesOrderHeader(SalesOrderID, CustomerID, OrderDate, TotalDue)
- SalesOrderDetail(SalesOrderID, ProductID, OrderQty, LineTotal)
- Product(ProductID, Name, ProductCategoryID)
- ProductCategory(ProductCategoryID, Name)

Relationships:
- Customer joins CustomerAddress on CustomerID
- CustomerAddress joins Address on AddressID
- Customer joins SalesOrderHeader on CustomerID
- SalesOrderHeader joins SalesOrderDetail on SalesOrderID
- SalesOrderDetail joins Product on ProductID
- Product joins ProductCategory on ProductCategoryID
"""
{% endhighlight %}

By including this schema context in the prompt:
- The model understands how the database is structured, what tables are available, and how they relate. 
- It can generate valid SQL queries that follow the correct join paths and column references.
- Users don’t need to know SQL or how the schema is designed — they just ask questions in plain English.

This approach keeps the solution lightweight and eliminates the need for more complex architectures making it ideal for projects like ours where the schema is stable and known ahead of time.

### Step-by-Step Implementation

With that out of the way, it’s time to roll up our sleeves and build the solution from the ground up. We’ll walk through each phase from setting up version controlled infrastructure to deploying a production ready, containerized FastAPI application powered by Azure OpenAI.

Each step aligns with modern DevOps principles, ensuring the solution is secure, reproducible, and scalable. Let’s get started.

- ### Set Up Git Repository

To align with core DevOps principles such as version control, collaboration, and change traceability, we'll use Terraform to create a Git repository in Github to store our code and configuration. 

{% highlight javascript %}
# Clone the git repository containing the terraform files for this project
git clone git@github.com:musana-engineering/sqlcopilot.git

// Login to Azure CLI and set the subscription to use
az login

// Set the following Environment Variables
export ARM_CLIENT_ID="your_client_id_here"
export ARM_CLIENT_SECRET="your_client_secret_here"
export ARM_TENANT_ID="your_tenant_id_here"
export ARM_SUBSCRIPTION_ID="your_subscription_id>

// Navigate to the github directory
cd sqlcopilot/infra/github
// 
{% endhighlight %}

- ### Provision Azure Infrastructure
- ### Build the API with FastAPI
- ### Dockerize the App
- ### Deploy to Azure Container Apps

### Conclusion


