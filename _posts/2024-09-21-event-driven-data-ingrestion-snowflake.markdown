---
layout: post
title: Event-Driven Data Ingestion into Snowflake - Powered by Azure Event Hub, Argo Events, and Argo Workflows
date: 2024-09-20 13:32:20 +0300
description: Exploring the practical implementation of an event-driven architecture for seamless data ingestion into Snowflake, utilizing Microsoft Azure External Stages, Event Hubs, Argo Events, and Argo Workflows.
img: snowflake.jpg # Add image post (optional)
fig-caption: # Add figcaption (optional)
tags: [platformengineering, kubernetes, snowflake, datapipelines]
---
Loading and processing data are crucial for any effective data analytics platform. Snowflake provides various methods for data ingestion, including bulk loading and continuous processing. In this article, I’ll dive into the practical implementation of the event-driven method.

Event-driven data ingestion loads data into a target system in response to specific events or triggers. This approach is generally more efficient than traditional batch processing since it allows for real-time responses to events.

In this article, I’ll walk you through setting up automated data ingestion whenever a new file is uploaded to a Microsoft Azure Blob storage account, which will serve as our named external stage for Snowflake. This method boosts efficiency and gives us access to the latest data in near real-time.
![architecture](https://github.com/user-attachments/assets/c37fd0af-aa9a-432c-9add-da76e24cb7a1)
## Table of Contents
- [Prerequisites ](#prerequisites)
- [Introducing JavaSips ](#introducing-java-sips)
  - [Ingestion Architecture Overview ](#ingestion-architecture-overview)
  - [Data Upload](#data-upload)
  - [Event Handling ](#event-handling)
  - [Workflow Execution ](#workflow-execution)
  - [Data Loading ](#data-loading)
- [Summary ](#summary)

## Prerequisites
Before you get started, please ensure you have the following:

- **Snowflake account:** This is where we will set up the Snowflake resources such as data warehouses, databases and tables to support the data ingestion processes discussed here. Sign up for a **[trial account](https://signup.snowflake.com/?utm_source=google&utm_medium=paidsearch&utm_campaign=na-us-en-brand-trial-exact&utm_content=go-eta-evg-ss-free-trial&utm_term=c-g-snowflake%20trial%20account-e&_bt=579123129595&_bk=snowflake%20trial%20account&_bm=e&_bn=g&_bg=136172947348&gclsrc=aw.ds&gad_source=1&gclid=Cj0KCQjw3bm3BhDJARIsAKnHoVWVpbV2-xagFD0LBmC-kxgnMcg0cH1afvWSLIko69Y0DtP6mnHRUCYaAjUREALw_wcB)**.
- **Azure account and subscription:** This is where we will set up the cloud infrastructure needed to support the data ingestion processes discussed here. Sign up for **[trial account](https://azure.microsoft.com/en-gb/pricing/offers/ms-azr-0044p/)**
- **[Terraform](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli)** installed on your local machine. You will need this when we get to the steps to provision the resources that we need to support the data ingestion processes discussed here. These will be provisioned in the Snowflake and Azure accounts.
- **[Kubectl](https://kubernetes.io/docs/tasks/tools/)** installed on your local machine. You'll need this to interact with a Kubernetes cluster when we get to the steps for setting up Argo Events and Argo Workflows.
- **Kubernetes cluster**. Ensure that Argo Events and Argo Workflows are installed and configured on that cluster. If you haven’t set them up yet, you can refer to my earlier article on this topic: **[Platform Engineering on Kubernetes](https://musana.engineering/platform-engineering-on-k8s-part1/)**

## Introducing JavaSips
**JavaSips** is a fictitious company dedicated to providing a range of coffee-derived products, including beverages and pastries. Their mission is to deliver the best coffee in the world on demand, ensuring swift service no matter where or when customers place their orders.

The production factories owned by JavaSips form a dense network, a mesh of coffee production units that spans several countries. At the end of each day, the operations team at each factory uploads inventory and order data files to an Azure Blob Storage account.

To improve operational efficiency, JavaSips aims to implement an event-driven architecture for data ingestion into their Snowflake account. This system will allow them to react promptly to the new files uploaded by each factory.

- ### Ingestion Architecture Overview
  - **Data Upload:** At the end of each day, each factory’s operations team uploads inventory and order data files to Azure Blob Storage, ensuring that all relevant data is centralized.
  - **Event Generation:** Each time a new data file is uploaded, a BlobCreated event is triggered in Azure Blob Storage. This event is then pushed using Azure Event Grid to an Event Hub subscriber. Event Grid uses event subscriptions to route event messages to subscribers. The image below illustrates the relationship between event publishers, event subscriptions, and event handlers.
![event-model](https://github.com/user-attachments/assets/46ce9471-493d-4f89-af9c-1ec910d314bc)
  - **Event Handling:** CafeJaba utilizes Azure Event Hubs to capture these BlobCreated events in real time, tracking every file upload efficiently across their global network.
  - **Workflow Execution:** The BlobCreated event is sent to Argo Events, which triggers an Argo workflow. Within the workflow steps, we extract the file URL from the incoming event, load the file into a Snowflake internal stage and then execute a COPY command to transfer the data from the internal stage into the factory specific Snowflake table.

Now that we have an example to work with, let’s see how we design this architecture for the JavaSips data platform.

## Next Section


## Summary
With this automated ingestion process, CafeJaba can analyze order trends and manage inventory in real time, allowing for rapid responses to customer demands. This enhances operational efficiency and elevates customer satisfaction.

