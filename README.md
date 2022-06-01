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

This repository is intended for Confluent technical staff to build and/or demonstrate. All of the steps and code needed are provided below. 

***

## Prerequisites

NOTE: The SnowflakeSink connector *must* deploy to the same cloud provider and region as Confluent Cloud. Running the database instances in the same provider/region will minimize latency and expense.

If you do not have a Confluent Cloud account, you can open one for free and get trial credits. No payment details are required. Use [this sign-up page](https://www.confluent.io/confluent-cloud/tryfree/) to get started. 

The fully-managed Snowflake connector may not work with a Snowflake free trial account. We have not tested this option.

We use Docker and Terraform to deploy Postgres and MySQL to the cloud. You do not need to modify the Docker images, but you should review your Terraform script for the correct region and other details. We provide details for deploying to:
* AWS `us-west-2` or `us-east-2`
* GCP _______________________
* Azure (future release)

Each cloud provider has its own requirements for integration as follows. We recommend avoiding production environments for this project.

- AWS
    - A user account with an API Key and Secret Key
    - Permissions to create various cloud resources
- GCP 
    - A project with permissions to create various cloud resources
    - A user account with a JSON Key file
- Azure
    - (future release)

To sink data to a data warehouse in real-time, we provide support for both Snowflake and Databricks. We do not offer fully-tested instructions in this repository. Follow the documentation available from either Databricks or Snowflake to create your accounts. The following are some key requirements:
- Snowflake *(Tested with AWS)*
    - Allocated in the same cloud provider/region as Confluent Cloud
    - We provide details for creating the database your connector writes to
    - To conserve Snowflake credits, we recommend you delay setting it up until you're ready to deploy the connector
- Databricks *(Tested with AWS)*
    - Allocated in the same provider/region as Confluent Cloud.
    - An S3 bucket in which the Delta Lake Sink Connector can stage data (explained in the link below).
    - Please review and walk through the following documentation to verify the appropriate setup within AWS and Databricks.
        - [Set Up Databricks Delta Lake](https://docs.confluent.io/cloud/current/connectors/cc-databricks-delta-lake-sink/databricks-aws-setup.html).

***

## Step-by-Step

### Project setup

1. Clone this repo to your computer.
    ```bash
    git clone https://github.com/zacharydhamilton/realtime-datawarehousing
    ```
    ```bash
    cd realtime-datawarehousing
    ```
1. Create an `env.sh` file to hold various settings throughout the setup. You derive a number of these values as you walk through the steps. The following file setup will save you a bit of typing:

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
    > **Note:** *Use `source env.sh` to read these values into your shell environment.*

1. Create a cluster in Confluent Cloud. A Basic or Standard type is sufficient.
    - [Create a Cluster in Confluent Cloud](https://docs.confluent.io/cloud/current/clusters/create-cluster.html).
    - Once you create a cluster, select its tile and click on **Cluster overview > Cluster settings**. Find the value for **Bootstrap server** and paste it to `BOOTSTRAP_SERVERS` in your `env.sh` file. 

1. Generate an API Key pair for authenticating to your Kafka cluster.
    - [Create API Keys](https://docs.confluent.io/cloud/current/access-management/authenticate/api-keys/api-keys.html#ccloud-api-keys).
    - Copy the values for the key and secret to `KAFKA_KEY` and `KAFKA_SECRET`, respectively, in your `env.sh` file. 

1. [Enable Schema Registry]((https://docs.confluent.io/cloud/current/get-started/schema-registry.html#enable-sr-for-ccloud)) 
    - Do this *before* you create a ksqlDB cluster.
    - Select the **Schema Registry** tab in your Confluent Cloud environment and locate the **API endpoint**. Paste this value into your `env.sh` file for `SCHEMA_REGISTRY_URL`.

1. [Create a Schema Registry API Key](https://docs.confluent.io/cloud/current/get-started/schema-registry.html#create-an-api-key-for-ccloud-sr).
    - Copy the values for the key and secret to `SCHEMA_REGISTRY_KEY` and `SCHEMA_REGISTRY_SECRET` respectively. 
1. [Create a ksqlDB cluster](https://docs.confluent.io/cloud/current/get-started/ksql.html#create-a-ksql-cloud-cluster-in-ccloud).
    - ksqlDB clusters that are created before the Schema Registry is enabled may not register with it correctly.

***

### Build cloud resources to host your source databases

The steps vary with each cloud provider. You can expand just the section you need for directions to your chosen cloud provider. Be sure to use the same provider/region your Confluent Cloud is using.

<details>
    <summary><b>AWS</b></summary>
    
1. Navigate to `terraform/aws`. If you are using an Apple M1 system, read `running-terraform-on-M1.md` for proper setup.

1. Navigate to `aws/` to initialize Terraform.
    
1. Review the `main.tf` file. You can copy `main.tf.us-west-2` or `main.tf.use-east-2` into `main.tf` before initializing.
    ```bash
    terraform init
    ```
1. Create a Terraform plan. You save the output to a file for use with `terraform apply`.
    ```bash
    terraform plan -out="<*region*>.plan"
    ```
1. Apply the plan to create the infrastructure.
    ```bash
    terraform apply <*region*>.plan
    ```

The `terraform apply` command prints the public IP addresses of the Postgres and Mysql instances it creates. You will need these later to configure your source connectors. Alternatively, you can locate the EC2 instances in your AWS account and get the details there.
    
You also can use `terraform show` once a plan has been applied to see the result.

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

The `terraform apply` command prints the public IP addresses of the Postgres and Mysql instances it creates. You will need these values to configure your source connectors. Alternatively, you can locate the instances in your GCP account and get the details there.
    
You also can use `terraform show` once a plan has been applied to see the result. 

</details>
<br>

<details>
    <summary><b>Azure</b></summary>

Coming Soon!

</details>
<br>

***

### Kafka Topics

1. In Confluent Cloud, create the topics your source connectors will write to. Navigate to the appropriate environment and cluster and click on `Topics` in the left-hand panel. Click the `Add Topic` button on the right-hand side of the display and add each of the following topic names. Set the `Partitions` field to 1 for each.
    - `postgres.products.products`
    - `postgres.products.orders`
    - `mysql.customers.customers`
    - `mysql.customers.demographics`

 ### Source Connectors
    
1. Return to the main cluster page. Expand the `Data Integration` in the left-hand menu and select `Connectors`. 
 
1. Locate and select the Debezium Postgres CDC Source Connector. Configure it with the following settings and launch it. 

    | **Property**                      | **Value**                          |
    |-----------------------------------|------------------------------------|
    | Kafka Cluster Authentication mode | KAFKA_API_KEY                      |
    | Kafka API Key                     | *copy from `env.sh`*               |
    | Kafka API Secret                  | *copy from `env.sh`*               |
    | Database hostname                 | *from Terraform output*            | 
    | Database port                     | `5432`                             |
    | Database username                 | `postgres`                         |
    | Database password                 | `rt-dwh-c0nflu3nt!`                |
    | Database name                     | `postgres`                         |
    | Database server name              | `postgres`                         |
    | Tables included                   | `products.products`, `products.orders` |
    | Output Kafka record value format  | `JSON_SR`                          |
    | Tasks                             | `1`                                |

    While the connector is provisioning, you can create the next one for MySQL. 

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
    | After-state only                  | `true`                                          |
    | Output Kafka record key format    | `JSON`                                          |
    | Tasks                             | `1`                                             |

Resolve any failures that occur. Once provisioned, the connectors will capturing data from the tables. 

> **Note:** *Only the `products.orders` table produces a running stream of records. The other produce records to their matching topics and stop. You will see low or no throughput, over time, for those topics. This is expected.*

<br>

***

### Ksql 

Next you will transform and join your data with Ksql. Be sure Schema Registry is enabled before continuing. 

1. Use the following statements to consume the `customers` data and flatten it for ease of use. 
    ```sql
        CREATE STREAM customers_structured (
            struct_key STRUCT<id VARCHAR> KEY,
            before STRUCT<id VARCHAR, first_name VARCHAR, last_name VARCHAR, email VARCHAR, phone VARCHAR>,
            after STRUCT<id VARCHAR, first_name VARCHAR, last_name VARCHAR, email VARCHAR, phone VARCHAR>,
            op VARCHAR
        ) WITH (
            KAFKA_TOPIC='mysql.customers.customers',
            KEY_FORMAT='JSON_SR',
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

1. With the `customers` data flattened, it can be easily aggregated into a Ksql table to retain the most up-to-date values by customer.
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

1. Next, do what is effectively the same thing, this time for the `demographics` data. 
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

1. And now create a Ksql table to retain the most up-to-date values by demographics. 
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

1. With the two tables `customers` and `demographics` created, they can be joined together to create what will effectively be an always up-to-date view of the customer and demographic data combined together by the customer ID. 
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

1. Next, use the following statements to capture the `products` data and re-key it.
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

1. Just like you did with the customer data, create a Ksql table to retain the most up-to-date values for the `products` data. 
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

1. Next, replicate what you did above with the `orders` data. 
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

1. Finally, create a Ksql stream to join **all** the tables together to enrich the order data in real time. 
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
    > **Note:** *You need a stream here so you can hydrate your data warehouse continuously. 

You should now have a working Ksql topology. Select **Flow** in the Ksql dashboard to view it.

***

### Data Warehouse Connectors

With the data now being captured from the two source databases and transformed in real-time, you're ready to sink the data to your data warehouse with another connector. Expand the section below corresponding to the data warehousing technology of your choice, and follow the directions to set it up. 

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
