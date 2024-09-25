---
layout: post
title: Snowflake Data Pipelines on Kubernetes- An Event-Driven Approach with Argo Events and Argo Workflows
date: 2024-09-20 13:32:20 +0300
description: A practical implementation of an event-driven architecture for seamless data ingestion into Snowflake, utilizing Microsoft Azure External Stages, Event Hubs, Argo Events, and Argo Workflows.
img: snowflake.jpg # Add image post (optional)
fig-caption: # Add figcaption (optional)
tags: [platformengineering, kubernetes, snowflake, datapipelines, argo, argo-events]
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
- **Azure account:** This is where we will set up the cloud infrastructure needed to support the data ingestion processes discussed here. Sign up for **[trial account](https://azure.microsoft.com/en-gb/pricing/offers/ms-azr-0044p/)**
- **Azure service principal:** To be used for Terraform provider authentication
- **[Terraform](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli):** You will need this to provision the Cloud and Snowflake resources we need to support the data ingestion processes discussed here.
- **[SnowSQL](https://developers.snowflake.com/snowsql/):** This is the command line client for connecting to Snowflake to execute SQL queries and perform all DDL and DML operations.
- **[Kubectl](https://kubernetes.io/docs/tasks/tools/):** You'll need this to interact with a Kubernetes cluster to create the Argo components.
- **[Kubernetes](https://kubernetes.io/):** A cluster with Argo Events and Argo Workflows installed. Checkout **[Part 1](https://musana.engineering/platform-engineering-on-k8s-part1/)** and **[Part 1](https://musana.engineering/platform-engineering-on-k8s-part2/)** of my series on this topic

### Introducing GloboLatte
**GloboLatte**, is a fictitious company that specializes in selling coffee-derived products like beverages and pastries. Their goal is to provide the best coffee, offering swift service regardless of when and where customers place their orders.

GloboLatte operates business units in America, Canada, and Mexico. At the end of each day, the operations team at each business unit uploads sales data files to an Azure Blob Storage account. To improve operational efficiency, GloboLatte aims to implement an event-driven architecture for data ingestion into their Snowflake account. This system will allow them to react promptly to the new sales data uploaded by each business unit.

To design an effective Snowflake database for GloboLatte, we’ll establish a structured schema that accommodates their sales data and optimizes for event-driven data ingestion. Below is a proposed design including database, schema, tables, and warehouses.

- ### Snowflake database design
  - **Database:**  GLOBO_LATTE_DB
  - **Warehouse:** GLOBO_LATTE_WH
  - **Schema:**    SALES_DATA
  - **File Format** CSV
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
The image below illustrates the relationship between event publishers, event subscriptions, and event handlers.
![eventModel](https://github.com/user-attachments/assets/765f405d-37f5-405c-83bd-796bae4193cf)
  - **Data upload:** At the end of each day, the sales operations team from each business unit uploads their sales data files to Azure Blob Storage for centralized access and analysis.
  - **Event generation:** Each time a new data file is uploaded, a BlobCreated event is triggered in Azure Blob Storage. This event is then pushed using Azure Event Grid to an Event Hub subscriber. Event Grid uses event subscriptions to route event messages to subscribers.
  - **Event handling:** GloboLatte utilizes Azure Event Hubs to capture these BlobCreated events in real time, tracking every file upload efficiently across their global network.
  - **Workflow execution:** The BlobCreated event are routed to Argo Events, triggering an Argo workflow. Within this workflow, we load the files into a Snowflake internal stage. Finally, we execute a COPY command to transfer the data from the internal stage into Snowflake tables

- ### Differences from Snowpipe
While Snowpipe is a powerful tool for continuous data ingestion into Snowflake, our kubernetes based approach offers several advantages, particularly in terms of cost efficiency and resource utilization.
  - **Cost Considerations:** Snowpipe can be expensive at high data volumes due to its pricing model based on the amount of data processed and the frequency of loading.
  - **Kubernetes Integration:** By leveraging existing Kubernetes clusters, we take advantage of built-in mechanisms for cost savings. 
  - **Open-Source Flexibility:** Our solution employs Argo Events and Argo Workflows, both of which are open-source tools - hence eliminates ongoing licensing fees associated with Snowpipe, making our approach more budget-friendly for organizations with high ingestion needs.
  - **Enhanced Control:** Unlike Snowpipe, which can feel like a black box, our Kubernetes-based solution provides greater control and allows for extensive customization of the data ingestion process.

Now that we have an example scenario to work with and the benefits outlined above, let’s explore how we can implement this architecture for the GloboLatte data platform.

### Create the Azure components
This implementation will leverage a variety of cloud components, including Azure Storage and Azure Event Hub, alongside Snowflake resources such as databases, warehouses, and tables. All resources will be defined and provisioned using Terraform, ensuring a streamlined and efficient setup. To setup the foundation for our data ingestion platform, we'll start by deploying the necessary resources in Azure. 

Apply the terraform configuration to provision the resources.

{% highlight shell %}
// Clone the git repository containing the terraform files for this project
git clone https://github.com/musana-engineering/snowflake.git

// Navigate to the azure directory
cd snowflake/azure

// Login to Azure CLI and set the subscription to use
az login
az account set -s "your_subscription_id_here"

// Set the following Environment Variables
export ARM_CLIENT_ID="your_client_id_here"
export ARM_CLIENT_SECRET="your_client_secret_here"
export ARM_TENANT_ID="your_tenant_id_here"

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

{% highlight shell %}
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

{% highlight shell %}
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

### Create the Argo Events componets
With the necessary resources for our data ingestion pipeline established in Azure and Snowflake, let's now enhance our system by integrating event-driven capabilities through Argo Events and Argo Workflows. The goal is to listen for messages in Azure Event Hubs and trigger workflows in Argo based on these events, which will execute data ingestion commands using the SnowSQL CLI.

To accomplish this, we will set up the following resources:

- **[EventSource](https://argoproj.github.io/argo-events/concepts/event_source/):** The EventSource will define the configurations required to consume events from various external sources, transform them into CloudEvents and dispatch them to the EventBus. In our setup, the EventSource will be configured to consume events from Azure Event Hub.

{% highlight yaml %}
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

- **[EventBus](https://argoproj.github.io/argo-events/concepts/eventbus/):** The EventBus will serve as the transport layer for Argo Events, connecting our EventSource and Sensor. EventSources publish events, while Sensors subscribe to these events to execute corresponding triggers. In our setup, the Azure Event Hub EventSource will publish messages to the Argo Events EventBus

{% highlight yaml %}
apiVersion: argoproj.io/v1alpha1
kind: EventBus
metadata:
  name: eventbus
  namespace: argo-events
spec:
  nats:
    native:
      // Optional, defaults to 3. requirement.
      replicas: 3
      // authen strategy, "none" or "token", defaults to "none"
      auth: token
{% endhighlight %}

- **[Sensor](https://argoproj.github.io/argo-events/concepts/sensor/):** The Sensor wil define a set of event dependencies (inputs) and triggers (outputs). It will listen for events on the EventBus and acts as an event dependency manager, resolving and executing triggers as events are received. In our setup, the Sensor leverages the EventSource and EventBus as its dependencies.

{% highlight yaml %}
apiVersion: argoproj.io/v1alpha1
kind: Sensor
metadata:
  name: azure-events-hub
  namespace: argo-events
spec:
  eventBusName: eventbus
  template:
    serviceAccountName: sa-argo-workflow
  dependencies:
    - name: azure-events-hub
      // The EventSource this Sensor listens to
      eventSourceName: azure-events-hub
      // The EventBus to which the EventSource publishes events
      eventBusName: eventbus
      // The specific event name to listen for
      eventName: snowflake
  ---
// Rest of the parts removed for Brevity //
{% endhighlight %}

Connect to your Kubernetes cluster and create the resources following the steps below:

{% highlight shell %}
// Navigate to the folder containing the argo configuration
cd snowflake/argo

// Create the EventBus
kubectl apply -f eventbus.yaml

// Create the EventSource
kubectl apply -f eventsource.yaml

// Create the Sensor
kubectl apply -f sensor.yaml
{% endhighlight %}

After deploying the resources, verify that they have been successfully created by running the following commands:

{% highlight shell %}
// Set the current context to the argo-events namespace
kubectl config set-context --current --namespace=argo-events

// Retrieve and display the EventBus, EventSource, and Sensor resources
kubectl get EventBus && kubectl get EventSource && kubectl get Sensor
{% endhighlight %}

### Create the Argo Worklow componets

- **[Workflow](https://argo-workflows.readthedocs.io/en/release-3.5/workflow-concepts/):** The Workflow defines the SnowSQL steps needed to transfer files from the named external stage (in this case, our Azure Blob storage account) into our Snowflake internal stage, and subsequently copy them into the tables. The Workflow is structured as a Directed Acyclic Graph (DAG) with the following steps for data ingestion:

  - Create a Named External Stage
  - Load the Data from the named external stage into a target table
  - Validate the Load by verifying that the rows were successfully loaded into the table 
  - Cleanup the Stage after confirming the successful data load
  
- **Setup Cloud Storage via External Stage**
Before creating the Workflow component which is the final piece of our ingestion pipeline, we need to configure Snowflake with an external stage backed by Microsoft Azure Cloud Storage. We can achieve this in three simple steps

{% highlight shell %}
// Step 1: Create a Cloud Storage Integration in Snowflake

CREATE STORAGE INTEGRATION azure_sagloballatter
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = 'AZURE'
  ENABLED = TRUE
  AZURE_TENANT_ID = 'your_azure_tenant_id>
  STORAGE_ALLOWED_LOCATIONS = 'azure://sagloballatte.blob.core.windows.net/';

// Step 2: Grant Snowflake Access to the Storage Locations

DESC STORAGE INTEGRATION $STORAGE_INTEGRATION_NAME;

// Step 3: validate the configuration for your storage integration

SELECT SYSTEM$VALIDATE_STORAGE_INTEGRATION('AZURE_SAGLOBALLATTE', 'azure://sagloballatte.blob.core.windows.net/america/sales_transaction/', 'vehicles.csv', 'read');

// Step 4: Create File Format to match the data file structure.

CREATE OR REPLACE FILE FORMAT CSV_With_Headers
  type = 'CSV'
  field_delimiter = ','
  skip_header = 1;
{% endhighlight %}

With that out of the way, lets define the worflow components

{% highlight yaml %}
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: snowflake-data-ingestion-
  namespace: argo-events
spec:
  entrypoint: pipeline
  serviceAccountName: sa-argo-workflow
  arguments:
    parameters:
      - name: body
        value: "placeholder"       
  templates:
    - name: pipeline
      dag: 
        tasks:  
          - name: create-external-stage
            template: create-external-stage
            arguments:
              parameters:
                - name: body
                  value: "{{workflow.parameters.body}}"
        
          - name: load-data
            template: load-data
            dependencies:
              - create-external-stage
          
          - name: validate-load
            template: validate-load
            dependencies:
              - load-data

          - name: cleanup-stage
            template: cleanup-stage
            dependencies:
              - validate-load

    - name: create-external-stage
      serviceAccountName: sa-argo-workflow   
      script:
        imagePullPolicy: "Always"
        image: musanaengineering/platformtools:snowsql:1.0.0
        command: [/bin/bash]
        source: |

          echo "Creating named external stage SALES_TRANSACTIONS..."
          snowsql -q "SHOW DATABASES;"

          snowsql -q "
          USE SCHEMA GLOBO_LATTE_DB.SALES_DATA;
          CREATE STAGE sales_transactions
            STORAGE_INTEGRATION = azure_blob_storage
            URL = 'azure://sagblatte.blob.core.windows.net/sfingestion/'
            FILE_FORMAT = CSV_FORMAT;"

    - name: load-data
      serviceAccountName: sa-argo-workflow   
      script:
        imagePullPolicy: "Always"
        image: musanaengineering/platformtools:snowsql:1.0.0
        command: [/bin/bash]
        source: |

          echo "Creating named external stage SALES_TRANSACTIONS..."
          snowsql -q "SHOW DATABASES;"

          snowsql -q "
          USE SCHEMA GLOBO_LATTE_DB.SALES_DATA;
          CREATE STAGE sales_transactions
            STORAGE_INTEGRATION = azure_blob_storage
            URL = 'azure://sagblatte.blob.core.windows.net/sfingestion/'
            FILE_FORMAT = CSV_FORMAT;"

    - name: validate-load
      serviceAccountName: sa-argo-workflow   
      script:
        imagePullPolicy: "Always"
        image: musanaengineering/platformtools:snowsql:1.0.0
        command: [/bin/bash]
        source: |
          
          echo "Creating named external stage SALES_TRANSACTIONS..."
          snowsql -q "SHOW DATABASES;"

    - name: cleanup-stage
      serviceAccountName: sa-argo-workflow   
      script:
        imagePullPolicy: "Always"
        image: musanaengineering/platformtools:snowsql:1.0.0
        command: [/bin/bash]
        source: |
          
          echo "Show tables in the global database"
          snowsql -q "
          USE DATABASE GLOBO_LATTE_DB;
          USE SCHEMA SALES_DATA;
          SHOW TABLES;"
{% endhighlight %}

Connect to your Kubernetes cluster and create the resources following the steps below:
{% highlight shell %}
// Navigate to the folder containing the argo configuration
cd snowflake/argo

// Create the Workflow
kubectl apply -f workflow.yaml
{% endhighlight %}


### Summary
With this automated ingestion process, GloboLatte can analyze order trends and manage inventory in real time, allowing for rapid responses to customer demands. This enhances operational efficiency and elevates customer satisfaction..

### NOTE: This article is NOT finished and still under development.......
