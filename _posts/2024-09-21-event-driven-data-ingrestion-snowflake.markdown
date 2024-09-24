---
layout: post
title: Snowflake Data Ingestion Pipelines on Kubernetes - An Event-Driven Approach powered by Argo Events and Argo Workflows
date: 2024-09-20 13:32:20 +0300
description: Exploring the practical implementation of an event-driven architecture for seamless data ingestion into Snowflake, utilizing Microsoft Azure External Stages, Event Hubs, Argo Events, and Argo Workflows.
img: snowflake.jpg # Add image post (optional)
fig-caption: # Add figcaption (optional)
tags: [platformengineering, kubernetes, snowflake, datapipelines]
---
Event-driven data ingestion loads data into a target system in response to specific events or triggers. This approach is generally more efficient than traditional batch processing since it allows for real-time responses to events. **Snowflake** provides various methods for data ingestion including bulk loading and continuous processing. It allows the loading of data from the following cloud services:

- Amazon S3
- Google Cloud Storage
- Microsoft Azure Blob Storage.

In this article, we’ll explore a practical setup in which data ingestion is triggered by new file uploads to a Microsoft Azure Blob Storage account, which will function as our external stage for Snowflake. The diagram below illustrates the architecture we’ll be building:

![architecture](https://github.com/user-attachments/assets/c37fd0af-aa9a-432c-9add-da76e24cb7a1)

### Table of Contents
- [Prerequisites ](#prerequisites)
- [Introducing GloboLatte ](#introducing-globolatte)
   - [Snowflake database design ](#snowflake-database-design)
   - [Ingestion architecture overview ](#ingestion-architecture-overview)
     - [Data upload](#data-upload)
     - [Event generation ](#event-generation)
     - [Event handling ](#event-handling)
     - [Workflow execution ](#workflow-execution)
   - [Differences from Snowpipe ](#differences-from-snowpipe)
- [Create the Azure components ](#create-the-azure-components)
- [Create the Snowflake components ](#create-the-snowflake-components)
- [Create the Argo components ](#create-the-argo-components)
- [Summary ](#summary)

### Prerequisites
Before you get started, please ensure you have the following:

- **Snowflake account:** This is where we will set up the Snowflake resources such as data warehouses, databases and tables to support the data ingestion processes discussed here. Sign up for a **[trial account](https://signup.snowflake.com/?utm_source=google&utm_medium=paidsearch&utm_campaign=na-us-en-brand-trial-exact&utm_content=go-eta-evg-ss-free-trial&utm_term=c-g-snowflake%20trial%20account-e&_bt=579123129595&_bk=snowflake%20trial%20account&_bm=e&_bn=g&_bg=136172947348&gclsrc=aw.ds&gad_source=1&gclid=Cj0KCQjw3bm3BhDJARIsAKnHoVWVpbV2-xagFD0LBmC-kxgnMcg0cH1afvWSLIko69Y0DtP6mnHRUCYaAjUREALw_wcB)**.
- **Azure account and subscription:** This is where we will set up the cloud infrastructure needed to support the data ingestion processes discussed here. Sign up for **[trial account](https://azure.microsoft.com/en-gb/pricing/offers/ms-azr-0044p/)**
- **[Terraform](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli)** installed on your local machine. You will need this when we get to the steps to provision the resources that we need to support the data ingestion processes discussed here. These will be provisioned in the Snowflake and Azure accounts.
- **[SnowSQL](https://developers.snowflake.com/snowsql/)** installed on your local machine. This is the command line client for connecting to Snowflake to execute SQL queries and perform all DDL and DML operations.
- **[Kubectl](https://kubernetes.io/docs/tasks/tools/)** installed on your local machine. You'll need this to interact with a Kubernetes cluster when we get to the steps for setting up Argo Events and Argo Workflows.
- **Kubernetes cluster**. Ensure that Argo Events and Argo Workflows are installed and configured on that cluster. If you haven’t set them up yet, you can refer to my earlier article on this topic: **[Platform Engineering on Kubernetes](https://musana.engineering/platform-engineering-on-k8s-part1/)**

### Introducing GloboLatte
**GloboLatte**, is a fictitious company that specializes in selling coffee-derived products like beverages and pastries. Their goal is to provide the best coffee globally, offering swift service regardless of when and where customers place their orders.

GloboLatte operates business units across several countries in America, South America, and Africa. At the end of each day, the operations team at each business unit uploads sales data files to an Azure Blob Storage account

To improve operational efficiency, GloboLatte aims to implement an event-driven architecture for data ingestion into their Snowflake account. This system will allow them to react promptly to the new sales data uploaded by each business unit.

To design an effective Snowflake database for GloboLatte, we’ll establish a structured schema that accommodates their sales data and optimizes for event-driven data ingestion. Below is a proposed design including database, schema, tables, and warehouses.

- ### Snowflake database design
  - **Database:**  GLOBO_LATTE_DB
  - **Warehouse:** GLOBO_LATTE_WH
  - **Schema:**    SALES_DATA
{% highlight ruby %}
- Sales_Transactions table

| Column Name        | Data Type | Description                                     |
|--------------------|-----------|-------------------------------------------------|
| `transaction_id`   | STRING    | Unique identifier for each transaction          |
| `business_unit`    | STRING    | Identifier for the business unit                |
| `product_id`       | STRING    | Identifier for the product sold                 |
| `customer_id`      | STRING    | Identifier for the customer                     |
| `quantity`         | INTEGER   | Number of items sold                            |
| `total_price`      | FLOAT     | Total price of the transaction                  |
| `transaction_date` | TIMESTAMP | Date and time of the transaction                |
| `payment_method`   | STRING    | Method of payment used (e.g., credit card)      |

- Products table

| Column Name        | Data Type | Description                                     |
|------------------|-----------  |-------------------------------------------------|
| `product_id`       | STRING    | Unique identifier for each product              |
| `product_name`     | STRING    | Name of the product                             |
| `category`         | STRING    | Category of the product                         |
| `price`            | FLOAT     | Price of the product                            |
| `stock_quantity`   | INTEGER   | Number of items available in stock              |

- Customers table

| Column Name        | Data Type | Description                                     |
|-----------------  -|-----------|-------------------------------------------------|
| `customer_id`      | STRING    | Unique identifier for each customer             |
| `customer_name`    | STRING    | Name of the customer                            |
| `email`            | STRING    | Email address of the customer                   |
| `location`         | STRING    | Geographical location of the customer           |

- Business_Units table

| Column Name        | Data Type | Description                                     |
|--------------------|-----------|-----------------------------------------        |
| `business_unit_id` | STRING    | Unique identifier for each business unit        |
| `country`          | STRING    | Country where the business unit operates        |
| `unit_name`        | STRING    | Name of the business unit                       |
{% endhighlight %}

- ### Ingestion architecture overview
![eventModel](https://github.com/user-attachments/assets/765f405d-37f5-405c-83bd-796bae4193cf)
  - **Data upload:** At the end of each day, the sales operations team from each business unit uploads their sales data files to Azure Blob Storage for centralized access and analysis.
  - **Event generation:** Each time a new data file is uploaded, a BlobCreated event is triggered in Azure Blob Storage. This event is then pushed using Azure Event Grid to an Event Hub subscriber. Event Grid uses event subscriptions to route event messages to subscribers. The image below illustrates the relationship between event publishers, event subscriptions, and event handlers.
  - **Event handling:** GloboLatte utilizes Azure Event Hubs to capture these BlobCreated events in real time, tracking every file upload efficiently across their global network.
  - **Workflow execution:** The BlobCreated event is routed to Argo Events, triggering an Argo workflow. Within this workflow, we first extract the file URL from the incoming event, then load the file into a Snowflake internal stage. Finally, we execute a COPY command to transfer the data from the internal stage into the specific Snowflake table for each factory.

- ### Differences from Snowpipe
While Snowpipe is a powerful tool for continuous data ingestion into Snowflake, our approach offers several advantages, particularly in terms of cost efficiency and resource utilization.
  - Snowpipe: While effective, Snowpipe can become expensive, especially at high data volumes due to its pricing model based on the amount of data processed and the frequency of loading.
  - By leveraging existing Kubernetes clusters, we can take advantage of built-in mechanisms for cost savings. 
  - Additionally, since Argo Events and Argo Workflows are open-source, our solution avoids the ongoing costs associated with Snowpipe, making it a more budget-friendly option for organizations with high ingestion needs.

Now that we have an example to work with, let’s see how we implement this architecture for the GloboLatte data platform.

### Create the Azure components

To setup the foundation for our data ingestion platform, we'll start by deploying the necessary resources in Azure. The resources are defined and provisioned by Terraform. 

If you havent already done so, create an **[Azure service principal](https://learn.microsoft.com/en-us/cli/azure/azure-cli-sp-tutorial-1?tabs=bash)** to be used for Terraform provider authentication. Ensure the service principal has been assigned atleast the **Contributor** role on your Azure subscription.

Next, Apply the terraform configuration to provision the resources.

{% highlight ruby %}
// Login to Azure CLI and set the subscription to use
az login
az account set -s "your_subscription_id_here"

// Set the following Environment Variables
export ARM_CLIENT_ID="your_client_id_here"
export ARM_CLIENT_SECRET="your_client_secret_here"
export ARM_TENANT_ID="your_tenant_id_here"

// Clone the project repository
git clone https://github.com/musana-engineering/snowflake.git

// Navigate to the azure directory
cd snowflake/azure

// Generate and review the Terraform plan
terraform init && terraform plan

// Provision the resources.
terraform apply
{% endhighlight %}

Here’s a breakdown of what gets created:
- **Dedicated Resource group** is created to organize and manage all related resources.
- **Virtual Network** 
    - Established with a defined address space specified
    - A core subnet is created within the virtual network
    - Service endpoints are configured for both Azure Storage and Azure Event Hubs, for secure access.
- **Azure Event Hub namespace** 
    - Trusted service access is enabled, and default action is set to "Deny," ensuring a secure environment. 
    - Specific IP rules are established to allow access from designated IP addresses.
    - A virtual network rule is configured to enable access from the specified subnet.
    - A system-assigned identity is created for secure access management.
    - A Private endpoint is created, linking the namespace to a specific subnet, isolating it from the public internet
    - A private DNS zone group is configured, ensuring that DNS resolution for the private endpoint works seamlessly.
    - An Event Hub named "snowflake" is created within the namespace.
- **Event Grid system topic** is set up to connect the Azure Storage account.
    - This topic is where BlobCreated events will originate. 
    - This topic facilitates event routing from the storage account to the Event Hub.
    - A system-assigned identity is also configured for secure interactions.
    - A role assignment is made, granting the "Azure Event Hubs Data Sender" role to the Event Grid system topic.
- **Event subscription** is created for the Event Grid system topic, specifically configured to handle "BlobCreated" events. This subscription routes these events to the Event Hub, enabling real-time processing of new data files uploaded to Azure Blob Storage
- **Private Link Access** is configured for direct access to system topics and domains within the Event Grid, ensuring that events can be sent securely without exposing data to the public internet

### Create the Snowflake componets

To establish the necessary infrastructure in Snowflake for GloboLatte’s data ingestion and analysis, let's execute our Terraform code for the same.

{% highlight ruby %}
// Set the following Environment Variables
export TF_VAR_account_name="your_snowflake_account_name"
export TF_VAR_account_username="your_snowflake_username"
export TF_VAR_account_password="your_snowflake_password"

// Navigate to the snowflakes directory
cd snowflake/snowflake

// Generate and review the Terraform plan
terraform init && terraform plan

// Provision the resources.
terraform apply
{% endhighlight %}

Here’s a breakdown of what gets created:
- **Database** named GLOBO_LATTE_DB. This serves as the primary container for all data objects, including schemas, tables, and file formats related to GloboLatte's operations.
Schema: SALES_DATA
- **Schema** schema called SALES_DATA within the GLOBO_LATTE_DB databse. Schemas help organize and group related tables logically, providing a clear structure for managing data assets associated with sales transactions and other related information.
- **Warehouse** named GLOBO_LATTE_WH with a size set to small. This warehouse will be utilized for processing queries, loading data, and running analytics. The small size is suitable for initial workloads, with the ability to scale as needed.
- **Tables:**
   - Sales transactions table to store transaction records.
   - Products table to hold product details
   - Customer table to store information about customers
   - Business units table to hold records of different business units
- **File format** named CSV_FORMAT which specifies how CSV files will be handled when loaded into Snowflake. The format configuration includes settings such as field delimiters, skipping headers, handling blank lines, and compression settings. This prepares the environment for seamless data ingestion from CSV files.

You can verify that all componets have been created using the SnowSQL CLI commands below

{% highlight ruby %}
// Set the following Environment Variables
export SNOWFLAKE_ACCOUNT="your_snowflake_account"
export SNOWFLAKE_USERNAME="your_snowflake_username"
export SNOWFLAKE_PASSWORD="your_snowflake_password"

// Login to Snowflake using SnowSQL
snowsql -a "$SNOWFLAKE_ACCOUNT" -u "$SNOWFLAKE_USERNAME" -P

// List Databases
SHOW DATABASES;
// List Warehouses
SHOW WAREHOUSES;
// List Tables
SHOW TABLES;
// List Table formats
SHOW FILE FORMATS;
{% endhighlight %}

### Create the Argo componets
With the necessary resources for our data ingestion pipeline established in Azure and Snowflake, let's now enhance our system by integrating event-driven capabilities through Argo Events and Argo Workflows. The goal is to listen for messages in Azure Event Hubs and trigger workflows in Argo based on these events, which will execute data ingestion commands using the SnowSQL CLI.

To accomplish this, we will set up the following resources:

- **EventSource:** An EventSource defines the configurations required to consume events from various external sources, such as AWS SNS, SQS, GCP Pub/Sub, and webhooks. It transforms incoming events into CloudEvents and dispatches them to the EventBus. In our setup, the EventSource will be configured to consume events from Azure Event Hub.

{% highlight javascript %}
apiVersion: argoproj.io/v1alpha1
kind: EventSource
metadata:
  name: azure-events-hub
  namespace: argo-events
spec:
  eventBusName: eventbus-main
  azureEventsHub:
    ceplatform:
      // FQDN of the EventsHub namespace you created
      fqdn: evhn-gblatte.servicebus.windows.net
      sharedAccessKeyName:
        name: secret_containing_shared_access_key_name
        key: key_within_the_secret_which_holds_the_value_of_shared_access_key_name
      sharedAccessKey:
        name: secret_containing_shared_access_key
        key: key_within_the_secret_which_holds_the_value_of_shared_access_key
      // Event Hub path/name
      hubName: salesdata
      jsonBody: true
{% endhighlight %}

- **EventBus:** The EventBus serves as the transport layer for Argo Events, connecting EventSources and Sensors. EventSources publish events, while Sensors subscribe to these events to execute corresponding triggers. In our setup, the Azure Event Hub EventSource will publish messages to the EventBus

{% highlight javascript %}
apiVersion: argoproj.io/v1alpha1
kind: EventBus
metadata:
  name: azure-eventhub
  namespace: argo-events
spec:
  jetstream:
    version: 2.8.1
{% endhighlight %}

- **Sensor:** A Sensor defines a set of event dependencies (inputs) and triggers (outputs). It listens for events on the EventBus and acts as an event dependency manager, resolving and executing triggers as events are received. In our setup, the Sensor will listen for events from the EventBus and trigger workflows in Argo Workflows 

Connect to your Kubernetes cluster and create the resources following the steps below:

{% highlight ruby %}
// Connect to your Kubernetes Cluster
export CLUSTER_NAME="your_snowflake_account"
export CLUSTER_RESOURCE_GROUP="your_snowflake_username"

az aks get-credentials -n $CLUSTER_NAME -g $CLUSTER_RESOURCE_GROUP

// Navigate to the snowflakes directory
cd snowflake/snowflake

// Create the EventBus
kubectl apply -f eventbus.yaml

// Create the EventSource
kubectl apply -f eventsource.yaml

// Create the Sensor
kubectl apply -f sensor.yaml
{% endhighlight %}

After deploying the resources, verify that they have been successfully created by running the following commands:

{% highlight ruby %}
kubectl get EventBus && \
kubectl get EventSource && \
kubectl get Sensor \
--namespace argo-events
{% endhighlight %}

### Summary
With this automated ingestion process, GloboLatte can analyze order trends and manage inventory in real time, allowing for rapid responses to customer demands. This enhances operational efficiency and elevates customer satisfaction..

### NOTE: This article is NOT finished and still under development.......
