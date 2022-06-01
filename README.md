<div align="center" padding=25px>
    <img src="images/confluent.png" width=50% height=50%>
</div>

# <div align="center">Real-time Data Warehouse Ingestion with Confluent Cloud</div>
## <div align="center">Workshop & Lab Guide</div>

## Background

This workshop and lab guide walks you through an ETL-style flow. It connects two external data sources to Confluent Cloud, joins datasets to produce  enriched customer records, then writes the record to a data warehouse as they are produced.

The components used to run this lab include:

- A Postgres database (`products`) running in a cloud environment
- A MySQL database (`customers`) running in a cloud environment
- Confluent Connect to deploy fully-managed connectors
- Kafka topics
- Confluent Schema Registry
- KsqlDB for data transformations 
- A Snowflake database running in an active warehouse

This repository is intended for Confluent technical staff to build and/or demonstrate. All of the steps and code needed to support building Confluent are provided below. 

***

## Prerequisites

NOTE: The SnowflakeSink connector *must* be deployed in the same cloud provider/region as Confluent Cloud. You should also deploy your database instances to the same location to minimize latency and expense. Also, the fully-managed Snowflake connector may not work with a Snowflake free trial account. We have not tested this option.

If you do not have a Confluent Cloud account, you can create one for free and use trial credits to complete this project. No payment details are required. Use [this sign-up page](https://www.confluent.io/confluent-cloud/tryfree/) to get started. 


This project uses Docker and Terraform to deploy Postgres and MySQL to the cloud. You do not need to modify the Docker images, but you should review your Terraform script for the correct region and other details. We provide details for deploying to:
* AWS `us-west-2` or `us-east-2`
* GCP _______________________
* Azure (future release)

Each cloud provider has its own requirements to integrate with Confluent Cloud:

- AWS
    - A user account with an API Key and Secret Key
    - Permissions to create various cloud resources
- GCP 
    - A project with permissions to create various cloud resources
    - A user account with a JSON Key file
- Azure
    - (future release)

To sink continuous data to your warehouse, we provide support for both Snowflake and Databricks. Use their documentation to create trial accounts. The following are some key requirements:
- Snowflake *(Tested with AWS)*
    - This repo provides details for creating the database your connector will write to
    - To conserve your Snowflake credits, you can delay setting it up until it's time to deploy the connector
- Databricks *(Tested with AWS)*
    - Allocated in the same provider/region as Confluent Cloud.
    - An S3 bucket in which the Delta Lake Sink Connector can stage data (explained in the link below).
    - Review and the [following documentation](https://docs.confluent.io/cloud/current/connectors/cc-databricks-delta-lake-sink/databricks-aws-setup.html) to set things up for AWS and Databricks.

***

## Step-by-Step

### Project setup

1. Clone this repo to your computer.
    ```bash
    git clone https://github.com/Incubate-or-Intubate/realtime-datawarehousing
    ```
    ```bash
    cd realtime-datawarehousing
    ```
1. Create an `env.sh` file to hold the binding values you need for setup. You'll gather these values throughout the setup. You can use the following script on Mac to save some typing. On Windows Powershell, you can replace the `cat` command with `type`.

    ```bash
    cat << EOF > env.sh
    # Confluent Cloud settings
    export BOOTSTRAP_SERVERS="<insert>"
    export KAFKA_KEY="<insert>"
    export KAFKA_SECRET="<insert>"
    export SASL_JAAS_CONFIG="org.apache.kafka.common.security.plain.PlainLoginModule required username='$KAFKA_KEY' password='$KAFKA_SECRET';"
    export SCHEMA_REGISTRY_URL="<insert>"
    export SCHEMA_REGISTRY_KEY="<insert>"
    export SCHEMA_REGISTRY_SECRET="<insert>"
    export SCHEMA_REGISTRY_BASIC_AUTH_USER_INFO="$SCHEMA_REGISTRY_KEY:$SCHEMA_REGISTRY_SECRET"
    export BASIC_AUTH_CREDENTIALS_SOURCE="USER_INFO"

    # AWS Credentials (used by Terraform)
    export AWS_DEFAULT_REGION="<insert>"
    export AWS_ACCESS_KEY_ID="<insert>"
    export AWS_SECRET_ACCESS_KEY="<insert>"
    # May be needed if you use `aws-gimme-creds` 
    export AWS_SECURITY_TOKEN="<insert>"
 
    # GCP Credentials (used by Terraform)
    export TF_VAR_GCP_PROJECT="<insert>"
    export TF_VAR_GCP_CREDENTIALS="<insert>"

    # Databricks
    export DATABRICKS_SERVER_HOSTNAME="<insert>"
    export DATABRICKS_HTTP_PATH="<insert>"
    export DATABRICKS_ACCESS_TOKEN="<insert>"
    export DELTA_LAKE_STAGING_BUCKET_NAME="<insert>"
    
    # Snowflake
    export SNOWFLAKE........="<insert>"
    export SNOWFLAKE........="<insert>"
    ```
    > **Note:** *Mac users can run `source env.sh` to read these values into your current shell.*

1. Create a cluster in Confluent Cloud. A Basic or Standard type is sufficient.
    - [Create a Cluster in Confluent Cloud](https://docs.confluent.io/cloud/current/clusters/create-cluster.html).
    - Once created, select the cluster's tile and select **Cluster overview > Cluster settings**. Copy the value for **Bootstrap server** and paste it to `BOOTSTRAP_SERVERS` line in your `env.sh` file. 

1. Generate an API Key pair for authenticating to your Kafka cluster.
    - [Create API Keys](https://docs.confluent.io/cloud/current/access-management/authenticate/api-keys/api-keys.html#ccloud-api-keys)
    - Copy the values for the key and secret to `KAFKA_KEY` and `KAFKA_SECRET`, respectively, in your `env.sh` file. 

1. [Enable Schema Registry](https://docs.confluent.io/cloud/current/get-started/schema-registry.html#enable-sr-for-ccloud)
    - Do this *before* you create a ksqlDB cluster.
    - Select **Schema Registry** tab in your Confluent Cloud environment (lower left-hand corner). Copy the value for **API endpoint** to `SCHEMA_REGISTRY_URL` in your `env.sh` file.

1. [Create a Schema Registry API Key](https://docs.confluent.io/cloud/current/get-started/schema-registry.html#create-an-api-key-for-ccloud-sr).
    - Copy the values for the key and secret to `SCHEMA_REGISTRY_KEY` and `SCHEMA_REGISTRY_SECRET` respectively. 

1. [Create a ksqlDB cluster](https://docs.confluent.io/cloud/current/get-started/ksql.html#create-a-ksql-cloud-cluster-in-ccloud).
    - ksqlDB clusters that are created before Schema Registry is enabled may not bind to the service correctly.

***

### Build a cloud environment to host your source databases

These steps vary with the provider. Expand the section below that you need for directions.

<details>
    <summary><b>AWS</b></summary>
1. If you are using an Apple M1 laptop, follow the instructions in `terraform/running-terraform-on-M1.md` first.   

1. Navigate to `terraform/aws` and review the `main.tf` file. You can copy `main.tf.us-west-2` or `main.tf.use-east-2` into `main.tf` if you wish. For all other regions, you'll first have to create an AMI.
    ```bash
    terraform init
    ```
   
1. Create a Terraform plan and save the output to a file.
    ```bash
    terraform plan -out="<*region*>.plan"
    ```

1. Apply the plan to create the infrastructure.
    ```bash
    terraform apply <*region*>.plan
    ```

1. The `terraform apply` command will output the public IP addresses for the Postgres and Mysql instances it creates. Save these to help configure your source connectors. You can also find the EC2 instances in your AWS account and get their IPs or public DNS names from their property lists.
    
1. You can use `terraform show` in the directory, once a plan has been applied, to see the results.

</details>
<br>

<details>
    <summary><b>GCP</b></summary>

1. Navigate to the GCP directory for Terraform.
    ```bash
    cd terraform/gcp
    ```
1. Initialize Terraform within the directory.
    ```bash
    terraform init
    ```
1. Create the Terraform plan.
    ```bash
    terraform plan
    ```
1. Apply the plan and create the infrastructure.
    ```bash
    terraform apply
    ```

1. The `terraform apply` command prints the public IP addresses of the Postgres and Mysql instances it creates. You will need these values to configure your source connectors. Alternatively, you can locate the instances in your GCP account and get the details there.
    
1. You also can use `terraform show` once a plan has been applied to see the result. 

</details>
<br>

<details>
    <summary><b>Azure</b></summary>

Coming Soon!

</details>
<br>

***

### Kafka Topics

1. In Confluent Cloud, create the topics your source connectors will use. In your target environment/cluster, click on `Topics` in the left-hand panel. Click the `Add Topic` button (right-hand side) and add a topic name. Set the `Partitions` field to 1 for each topic:
    - `postgres.products.products`
    - `postgres.products.orders`
    - `mysql.customers.customers`
    - `mysql.customers.demographics`

 ### Source Connectors
    
1. Return to the main cluster page. Expand the `Data Integration` section and select `Connectors`. 
 
1. Locate and select the Debezium Postgres CDC Source Connector. Configure it with the following settings and launch it. 

    | **Property**                      | **Value**                              |
    |-----------------------------------|----------------------------------------|
    | Kafka Cluster Authentication mode | KAFKA_API_KEY                          |
    | Kafka API Key                     | *copy from `env.sh`*                   |
    | Kafka API Secret                  | *copy from `env.sh`*                   |
    | Database hostname                 | *from Terraform output*                | 
    | Database port                     | `5432`                                 |
    | Database username                 | `postgres`                             |
    | Database password                 | `rt-dwh-c0nflu3nt!`                    |
    | Database name                     | `postgres`                             |
    | Database server name              | `postgres`                             |
    | Tables included                   | `products.products`, `products.orders` |
    | Output Kafka record value format  | `JSON_SR`                              |
    | Tasks                             | `1`                                    |

    Let the connector provision. You can start creating the next one right away. 

1. Locate and select the Debezium Mysql CDC Source Connector. Configure it with the following settings and launch it. 

    | **Property**                      | **Value**                                       |
    |-----------------------------------|-------------------------------------------------|
    | Kafka Cluster Authentication mode | KAFKA_API_KEY                                   |
    | Kafka API Key                     | *copy from `env.sh`*                            |
    | Kafka API Secret                  | *copy from `env.sh`*                            |
    | Database hostname                 | *from `terraform apply` output*                 |
    | Database port                     | `3306`                                          |
    | Database username                 | `debezium`                                      |
    | Database password                 | `rt-dwh-c0nflu3nt!`                             |
    | Database server name              | `mysql`                                         |
    | Databases included                | `customers`                                     |
    | Tables included                   | `customers.customers`, `customers.demographics` |
    | Output Kafka record value format  | `JSON_SR`                                       |
    | After-state only                  | `false`                                         |
    | Output Kafka record key format    | `JSON`                                          |
    | Tasks                             | `1`                                             |

Resolve any failures that occur. Once provisioned, the connectors will automatically receive data from each table. 

> **Note:** *Only the `products.orders` table produces a running stream of records. The others will produce records to their topics and stop. Over time, these connections will eventually report no throughput. This is expected.*

<br>

***

### Ksql 

Next you will transform and join your data. 

1. First, consume the `customers` data and flatten it. Note the records are first "de"-structed from its CDC before/after form.
    ```sql
        CREATE STREAM customers_structured (
            struct_key STRUCT<id VARCHAR> KEY,
            before STRUCT<id VARCHAR, first_name VARCHAR, last_name VARCHAR, email VARCHAR, phone VARCHAR>,
            after STRUCT<id VARCHAR, first_name VARCHAR, last_name VARCHAR, email VARCHAR, phone VARCHAR>,
            op VARCHAR
        ) WITH (
            KAFKA_TOPIC='mysql.customers.customers',
            KEY_FORMAT='JSON',
            VALUE_FORMAT='JSON_SR'
        );
    ```
    ```sql
        CREATE STREAM customers_flattened WITH (
                KAFKA_TOPIC='customers_flattened',
                KEY_FORMAT='JSON',
                VALUE_FORMAT='JSON_SR'
            ) AS SELECT
                after->id,
                after->first_name first_name, 
                after->last_name last_name,
                after->email email,
                after->phone phone
            FROM customers_structured
            PARTITION BY after->id
        EMIT CHANGES;
    ```

1. Next you will collect records into a table to keep only the latest values per customer record.
    ```sql
        CREATE TABLE customers WITH (
                KAFKA_TOPIC='customers',
                KEY_FORMAT='JSON',
                VALUE_FORMAT='JSON_SR'
            ) AS SELECT
                id,
                LATEST_BY_OFFSET(first_name) first_name, 
                LATEST_BY_OFFSET(last_name) last_name,
                LATEST_BY_OFFSET(email) email,
                LATEST_BY_OFFSET(phone) phone
            FROM customers_flattened
            GROUP BY id
        EMIT CHANGES;
    ```

1. Repeat the process above for your `demographics` records. 
    ```sql
        CREATE STREAM demographics_structured (
            struct_key STRUCT<id VARCHAR> KEY,
            before STRUCT<id VARCHAR, street_address VARCHAR, state VARCHAR, zip_code VARCHAR, country VARCHAR, country_code VARCHAR>,
            after STRUCT<id VARCHAR, street_address VARCHAR, state VARCHAR, zip_code VARCHAR, country VARCHAR, country_code VARCHAR>,
            op VARCHAR
        ) WITH (
            KAFKA_TOPIC='mysql.customers.demographics',
            KEY_FORMAT='JSON',
            VALUE_FORMAT='JSON_SR'
        );
    ```
    ```sql
        CREATE STREAM demographics_flattened WITH (
                KAFKA_TOPIC='demographics_flattened',
                KEY_FORMAT='JSON',
                VALUE_FORMAT='JSON_SR'
            ) AS SELECT
                after->id,
                after->street_address,
                after->state,
                after->zip_code,
                after->country,
                after->country_code
            FROM demographics_structured
            PARTITION BY after->id
        EMIT CHANGES;
    ```

1. Use a Ksql table to keep only ther latest values for `demographics` records. 
    ```sql
        CREATE TABLE demographics WITH (
                KAFKA_TOPIC='demographics',
                KEY_FORMAT='JSON',
                VALUE_FORMAT='JSON_SR'
            ) AS SELECT
                id, 
                LATEST_BY_OFFSET(street_address) street_address,
                LATEST_BY_OFFSET(state) state,
                LATEST_BY_OFFSET(zip_code) zip_code,
                LATEST_BY_OFFSET(country) country,
                LATEST_BY_OFFSET(country_code) country_code
            FROM demographics_flattened
            GROUP BY id
        EMIT CHANGES;
    ```

1. Next you will join these tables to create an up-to-the-miute view of customer and demographic data, joined by the customer's ID. 
    ```sql
        CREATE TABLE customers_enriched WITH (
                KAFKA_TOPIC='customers_enriched',
                KEY_FORMAT='JSON',
                VALUE_FORMAT='JSON_SR'
            ) AS SELECT 
                c.id id, c.first_name first_name, c.last_name last_name, c.email email, c.phone phone,
                d.street_address street_address, d.state state, d.zip_code zip_code, d.country country, d.country_code country_code
            FROM customers c
                JOIN demographics d ON d.id = c.id
        EMIT CHANGES;
    ```

1. Now we'll capture our `products` data, convert the record key, then partition the records.
    ```sql
        CREATE STREAM products_composite (
            struct_key STRUCT<product_id VARCHAR> KEY,
            product_id VARCHAR,
            `size` VARCHAR,
            product VARCHAR,
            department VARCHAR,
            price VARCHAR,
            __deleted VARCHAR
        ) WITH (
            KAFKA_TOPIC='postgres.products.products',
            KEY_FORMAT='JSON',
            VALUE_FORMAT='JSON_SR'
        );
    ```
    ```sql
        CREATE STREAM products_rekeyed WITH (
                KAFKA_TOPIC='products_rekeyed',
                KEY_FORMAT='KAFKA',
                VALUE_FORMAT='JSON_SR'
            ) AS SELECT 
                product_id,
                `size`,
                product,
                department,
                price,
                __deleted deleted
            FROM products_composite
            PARTITION BY product_id
        EMIT CHANGES;
    ```

1. Again we'll need a Ksql table to retain up-to-date values for each product record. 
    ```sql 
        CREATE TABLE products WITH (
                KAFKA_TOPIC='products',
                KEY_FORMAT='JSON',
                VALUE_FORMAT='JSON_SR'
            ) AS SELECT 
                product_id,
                LATEST_BY_OFFSET(`size`) `size`,
                LATEST_BY_OFFSET(product) product,
                LATEST_BY_OFFSET(department) department,
                LATEST_BY_OFFSET(price) price,
                LATEST_BY_OFFSET(deleted) deleted
            FROM products_rekeyed
            GROUP BY product_id
        EMIT CHANGES;
    ```

1. We'll follow the same process with the `orders` data. 
    ```sql
        CREATE STREAM orders_composite (
            order_key STRUCT<`order_id` VARCHAR> KEY,
            order_id VARCHAR,
            product_id VARCHAR,
            customer_id VARCHAR,
            __deleted VARCHAR
        ) WITH (
            KAFKA_TOPIC='postgres.products.orders',
            KEY_FORMAT='JSON',
            VALUE_FORMAT='JSON_SR'
        );
    ```
    ```sql
        CREATE STREAM orders_rekeyed WITH (
                KAFKA_TOPIC='orders_rekeyed',
                KEY_FORMAT='KAFKA',
                VALUE_FORMAT='JSON_SR'
            ) AS SELECT
                order_id,
                product_id,
                customer_id,
                __deleted deleted
            FROM orders_composite
            PARTITION BY order_id
        EMIT CHANGES;
    ```

1. Finally, we create an `orders_enriched` stream that combines the three Ksql tables together. We'll feed this stream to our sink connector. 
    ```sql
        CREATE STREAM orders_enriched WITH (
                KAFKA_TOPIC='orders_enriched',
                KEY_FORMAT='JSON',
                VALUE_FORMAT='JSON_SR'
            ) AS SELECT 
                o.order_key `order_key`, o.order_id `order_id`,
                p.product_id `product_id`, p.`size` `size`, p.product `product`, p.department `department`, p.price `price`,
                c.id `id`, c.first_name `first_name`, c.last_name `last_name`, c.email `email`, c.phone `phone`,
                c.street_address `street_address`, c.state `state`, c.zip_code `zip_code`, c.country `country`, c.country_code `country_code`
            FROM orders_composite o
                JOIN products p ON o.product_id = p.product_id
                JOIN customers_enriched c ON o.customer_id = c.id
            PARTITION BY o.order_key  
        EMIT CHANGES;  
    ```
    > **Note:** *A stream is required to hydrate the data warehouse. 

You should now have a working Ksql topology. In the Ksql dashboard, you can select the `Flow` tab to review it.

***

### Data Warehouse Connectors

If everything is working as expected, you're ready to sink data to your warehouse. Expand the section below that corresponds to your chosen site and follow the directions. 

> **Note:** *If you skipped over the prerequisites, you'll need to address those to be able to do this part of the lab.* 

<details>
    <summary><b>Databricks</b></summary>

1. Start by locating the Databricks cluster's JDBC/ODBC connection details. After selecting your cluster, expand the section titled **Advanced**, and then select the **JDBC/ODBC** tab. On the following page, select and copy the values for **Server Hostname** and **HTTP Path** to your clipboard file. 
    > **Note:** *If you don't have an S3 bucket, an AWS Key/secret, or the Databricks Access token described from doing the prerequisites, create or gather these values. 

1. Start by creating the Databricks Delta Lake Sink Connector. Select **Data integration > Connectors** from the left-hand menu, then search for the connector. When you find its tile, select it and configure it with the following settings, then launch it.
    | **Property**                      | **Value**                  |
    |-----------------------------------|----------------------------|
    | Topics                            | `orders_enriched`          |
    | Kafka Cluster Authentication mode | KAFKA_API_KEY              |
    | Kafka API Key                     | *copy from clipboard file* |
    | Kafka API Secret                  | *copy from clipboard file* |
    | Delta Lake Host Name              | *copy from clipboard file* |
    | Delta Lake HTTP Path              | *copy from clipboard file* |
    | Delta Lake Token                  | *from the prerequisites*   |
    | Staging S3 Access Key ID          | *from the prerequisites*   |
    | Staging S3 Secret Access Key      | *from the prerequisites*   |
    | S3 Staging Bucket Name            | *from the prerequisites*   |
    | Tasks                             | 1                          |

1. With the connector provisioned, data should be being sent to a Delta Lake Table in real time. Create the following table so you can query the datasets. 
    ```sql
        CREATE TABLE orders_enriched (order_id STRING, 
                                    product_id STRING, size STRING, product STRING, department STRING, price STRING,
                                    id STRING, first_name STRING, last_name STRING, email STRING, phone STRING,
                                    street_address STRING, state STRING, zip_code STRING, country STRING, country_code STRING,
                                    partition INT) USING DELTA;
    ```

1. And finally, query the records!
    ```sql 
     SELECT * FROM default.orders_enriched;
    ```

At this point, you can play around to your hearts desire with the dataset in Databricks. To emphasize was you accomplished, try constructing some queries that combine the data from two tables originating in the different source databases to do something cool. *Hint, total revenue by state, or something.*

</details>

<br>

<details>
    <summary><b>Snowflake</b></summary>

Coming Soon!

</details>

<br>

***

## Cleanup

When you're done with this lab, **delete everything you provisioned** to spare future costs. 

### Confluent Cloud
During this lab you created the following resources, be sure to remove them when you're done with them.
- Ksql Cluster
- Delta Lake Sink Connector
- Postgres CDC Source Connector
- Mysql CDC Source Connector
- Kafka Cluster

### Terraform
To remove everything provisioned by Terraform in either AWS, GCP, or Azure, use the following command.
    ```bash
    terraform destroy
    ```

### Databricks and Snowflake
If you created instances of either Databricks and Snowflake solely for the purpose of this lab, remove them!

***

## Useful Links

Databricks
- [Confluent Cloud Databricks Delta Lake Sink](https://docs.confluent.io/cloud/current/connectors/cc-databricks-delta-lake-sink/cc-databricks-delta-lake-sink.html)
- [Databricks Setup on AWS](https://docs.confluent.io/cloud/current/connectors/cc-databricks-delta-lake-sink/databricks-aws-setup.html)
