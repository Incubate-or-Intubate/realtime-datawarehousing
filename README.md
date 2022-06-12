<div align="center" padding=25px>
    <img src="images/confluent.png" width=50% height=50%>
</div>

# <div align="center">Real-time Data Warehouse Ingestion with Confluent Cloud</div>
## <div align="center">Workshop & Lab Guide</div>

## Background

The idea behind this workshop/lab guide is to provide a complete walk through of an example application that connects multiple external data sources to Confluent, joins their datasets together into one, and writes the new events to a data warehouse in real-time.

The core Confluent Cloud components that will be used to accomplish this will be:
- Kafka Connect
- Kafka
- KsqlDB
- Schema Registry

You can use this repository to create your own demo. All of the steps and code needed are included. If something's missing, please let us know! 

***

## Prerequisites

Get a Confluent Cloud account if you don't have one. New accounts start with $400 in credits and do not require a credit card. [Get Started with Confluent Cloud for Free](https://www.confluent.io/confluent-cloud/tryfree/).

You'll need a couple tools that make setup go a lot faster. Install these first. 
- `git`
- Docker
- Terraform
    - Special instructions for Apple M1 users are [here](./terraform/running-terraform-on-M1.md)

This repo uses Docker and Terraform to deploy your source databases to a cloud provider. What you need for this tutorial varies with each provider.
- AWS
    - A user account (use a testing environment) with permissions to create resources
    - An API Key and Secret to access the account from Confluent Cloud
- GCP 
    - A test project in which you can create resources
    - A user account with a JSON Key file and permission to create resources
- Azure
    - Coming in a future version

To sink streaming data to your warehouse, we support Snowflake and Databricks. This repo assumes you can have set up either account and are familiar with the basics of using them.
- Snowflake
    - Your account must reside in the same region as your Confluent Cloud environment
- Databricks *(AWS only)*
    - Your account must reside in the same region as your Confluent Cloud environment
    - You'll need an S3 bucket the Delta Lake Sink Connector can use to stage data (detailed in the link below)
    - Review [Databricks' documentation to ensure proper setup](https://docs.confluent.io/cloud/current/connectors/cc-databricks-delta-lake-sink/databricks-aws-setup.html)

***

## Step-by-Step

### Confluent Cloud Components

1. Clone and enter this repo.
    ```bash
    git clone https://github.com/iincubate-or-intubate/realtime-datawarehousing
    cd realtime-datawarehousing
    ```

1. Create a file to manage all the values you'll need through the setup. 
    ```bash 
    cat << EOF > env.sh
    # Confluent Creds
    export BOOTSTRAP_SERVERS="<replace>"
    export KAFKA_KEY="<replace>"
    export KAFKA_SECRET="<replace>"
    export SASL_JAAS_CONFIG="org.apache.kafka.common.security.plain.PlainLoginModule required username='$KAFKA_KEY' password='$KAFKA_SECRET';"
    export SCHEMA_REGISTRY_URL="<replace>"
    export SCHEMA_REGISTRY_KEY="<replace>"
    export SCHEMA_REGISTRY_SECRET="<replace>"
    export SCHEMA_REGISTRY_BASIC_AUTH_USER_INFO="$SCHEMA_REGISTRY_KEY:$SCHEMA_REGISTRY_SECRET"
    export BASIC_AUTH_CREDENTIALS_SOURCE="USER_INFO"
    
    # AWS Creds for TF
    export AWS_ACCESS_KEY_ID="<replace>"
    export AWS_SECRET_ACCESS_KEY="<replace>"
    export AWS_DEFAULT_REGION="us-east-2" # You can change this, but make sure it's consistent
    
    # GCP Creds for TF
    export TF_VAR_GCP_PROJECT=""
    export TF_VAR_GCP_CREDENTIALS=""
    
    # Databricks
    export DATABRICKS_SERVER_HOSTNAME="<replace>"
    export DATABRICKS_HTTP_PATH="<replace>"
    export DATABRICKS_ACCESS_TOKEN="<replace>"
    export DELTA_LAKE_STAGING_BUCKET_NAME="<replace>"
    
    # Snowflake
    SF_PUB_KEY="<replace>"
    SF_PVT_KEY="<replace>"
    EOF
    ```
    > **Note:** *Run `source env.sh` at any time to update these values in your terminal session. Do NOT commit this file to a GitHub repo.*

1. Create a cluster in Confluent Cloud. The Basic cluster type will suffice for this tutorial.
    - [Create a Cluster in Confluent Cloud](https://docs.confluent.io/cloud/current/clusters/create-cluster.html).
    - Select **Cluster overview > Cluster settings**. Paste the value for **Bootstrap server** into your `env.sh` file under `BOOTSTRAP_SERVERS`. 

1. [Create an API Key pair](https://docs.confluent.io/cloud/current/access-management/authenticate/api-keys/api-keys.html#ccloud-api-keys) for authenticating to the cluster.
    - Paste the values for the key and secret into `KAFKA_KEY` and `KAFKA_SECRET` in your `env.sh` file. 

1. [Enable Schema Registry](https://docs.confluent.io/cloud/current/get-started/schema-registry.html#enable-sr-for-ccloud)
    - Select the **Schema Registry** tab in your environment and locate **API endpoint**. Paste the endpoint value to your `env.sh` file under `SCHEMA_REGISTRY_URL`.

1. [Create an API Key for authenticating to Schema Registry](https://docs.confluent.io/cloud/current/get-started/schema-registry.html#create-an-api-key-for-ccloud-sr). 
    - Paste the key and secret into your `env.sh` file under `SCHEMA_REGISTRY_KEY` and `SCHEMA_REGISTRY_SECRET`. 

1. [Create a ksqlDB cluster](https://docs.confluent.io/cloud/current/get-started/ksql.html#create-a-ksql-cloud-cluster-in-ccloud).
    - Allow some time for this cluster to provision. This is a good opportunity to stand up and stretch.

***

### Build your cloud infrastructure

The next steps vary slightly for each cloud provider. Expand the appropriate section below for directions. Remember to specify the same region as your sink target!

<details>
    <summary><b>AWS</b></summary>

1. Navigate to the repo's AWS directory.
    ```bash
    cd terraform/aws
    ```
1. Initialize Terraform within the directory.
    ```bash
    terraform init
    ```
1. Create the Terraform plan.
    ```bash
    terraform plan -out=myplan
    ```
1. Apply the plan to create the infrastructure.
    ```bash
    terraform apply myplan
    ```

    > **Note:** *Read the `main.tf` configuration file [to see what will be created](./terraform/aws/main.tf).* 

The `terraform apply` command will print the public IP addresses of the host EC2 instances for your Postgres and Mysql services. You'll need these later to configuring the source connectors. 

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
    terraform plan --out=myplan
    ```
1. Apply the plan and create the infrastructure.
    ```bash
    terraform apply myplan
    ```
    > **Note:** *To see what resources are created by this command, see the [`main.tf` file here](https://github.com/incubate-or-intubate/realtime-datawarehousing/tree/main/terraform/gcp).

The `terraform apply` command will print the public IP addresses for the Postgres and Mysql instances it creates. You will need these to configure the connectors. 

</details>
<br>

<details>
    <summary><b>Azure</b></summary>
Coming Soon!
</details>
<br>

***

### Kafka Connectors

1. Create the topics that your source connectors need. Using the Topics menu, configure each onw with **1 partition** only.

    - `postgres.products.products`
    - `postgres.products.orders`
    - `mysql.customers.customers`
    - `mysql.customers.demographics`

1. Provision the Debezium Postgres CDC Source Connector. Select **Data integration > Connectors** from the left-hand menu, then search for the connector. Select the tile. Use the table below to configure it: 

    | **Property**                      | **Value**                          |
    |-----------------------------------|------------------------------------|
    | Kafka Cluster Authentication mode | `KAFKA_API_KEY`                    |
    | Kafka API Key                     | *get from `env.sh`*                |
    | Kafka API Secret                  | *get from `env.sh`*                |
    | Database hostname                 | *get from Terraform output*        |
    | Database port                     | `5432`                             |
    | Database username                 | `postgres`                         |
    | Database password                 | `rt-dwh-c0nflu3nt!`                |
    | Database name                     | `postgres`                         |
    | Database server name              | `postgres`                         |
    | Tables included                   | `products.products`, `products.orders` |
    | Output Kafka record value format  | `JSON_SR`                            |
    | Tasks                             | `1`                                  |

    Launch the connector. You can start on the MySQL connector while this one provisions.

1. Create the Debezium Mysql CDC Source Connect by searching for it and selecting its tile. Configure it using the following settings. 

    | **Property**                      | **Value**                                   |
    |-----------------------------------|---------------------------------------------|
    | Kafka Cluster Authentication mode | `KAFKA_API_KEY`                             |
    | Kafka API Key                     | *copy from `env.sh`*                        |
    | Kafka API Secret                  | *copy from `env.sh`*                        |
    | Database hostname                 | *get from Terraform output*                 |
    | Database port                     | `3306`                                      |
    | Database username                 | `debezium`                                  |
    | Database password                 | `rt-dwh-c0nflu3nt!`                         |
    | Database server name              | `mysql`                                     |
    | Databases included                | `customers`                                 |
    | Tables included                   | `customers.customers`, `customers.demographics` |
    | Output Kafka record value format  | `JSON_SR`                                   |
    | After-state only                  | `false`                                     |
    | Output Kafka record key format    | `JSON`                                      |
    | Tasks                             | `1`                                         |

Launch the connector. Once both are fully provisioned, check for and troubleshoot any failures that occur. Properly configured, each connector begins reading data automatically. 

> **Note:** *Only the `products.orders` table emits an ongoing stream of records. The others have their records produced to their topics from an initial snapshot only. After that, they do nothing more. The connector throughput will accordingly drop to zero over time.*

<br>

***

### Ksql 

If all is well, it's time to transform and join your data using Ksql. Ensure your topics are receiving records first.

1. Enter each of the following queries in the 

1. Use the following statements to consume `customers` records and flatten them. Note the `before` and `after` arrays representing CDC data.
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

1. With the `customers` records flattened, you can pass them into a Ksql table that updates the latest value provided for each field.
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

1. Repeat the process above for the `demographics` table. 
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

1. Create a Ksql table to present the the latest values by demographics. 
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

1. You can now join `customers` and `demographics` by customer ID to create am up-to-the-second view of each record. 
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

1. Next you will capture your `products` records and convert the record key to a simpler value.
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

1. Create a Ksql table to show the most up-to-date values for each `products` record. 
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

1. Follow the same process using the `orders` data. 
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

1. You're now ready to create a Ksql stream that joins these tables together to create enriched order data in real time. 
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
    > **Note:** *We need a stream to 'hydrate' our data warehouse once the sink connector is set up. 

Verify that you have a working Ksql topology. You can inspect it by selecting the **Flow** tab in the Ksql cluster. Check to see that records are populating the `orders_enriched` kstream.

***

### Data Warehouse Connectors

You're now ready to sink data to your chosen warehouse. Expand the appropriate section and follow the directions to set up your connector.  

<details>
    <summary><b>Databricks</b></summary>
    
1. Review the [source documentation](https://docs.confluent.io/cloud/current/connectors/cc-databricks-delta-lake-sink/cc-databricks-delta-lake-sink.html) if you prefer.

1. Locate your JDBC/ODBC details. Select your cluster. Expand the **Advanced** section and select the **JDBC/ODBC** tab. Paste the values for **Server Hostname** and **HTTP Path** to your `env.sh` file under `DATABRICKS_SERVER_HOSTNAME` and `DATABRICKS_HTTP_PATH`.
  
    > **Note:** *If you don't yet have an S3 bucket, AWS Key/secret, or Databricks Access token as described in the Prerequisites, create and/or gather them now. 

1. Create your Databricks Delta Lake Sink Connector. Select **Data integration > Connectors** from the left-hand menu and search for the connector. Select its tile and configure it using the following settings.
    
    | **Property**                      | **Value**                  |
    |-----------------------------------|----------------------------|
    | Topics                            | `orders_enriched`          |
    | Kafka Cluster Authentication mode | KAFKA_API_KEY              |
    | Kafka API Key                     | *copy from `env.sh` file*  |
    | Kafka API Secret                  | *copy from `env.sh` file*  |
    | Delta Lake Host Name              | *copy from `env.sh` file*  |
    | Delta Lake HTTP Path              | *copy from `env.sh` file*  |
    | Delta Lake Token                  | *from Databricks setup*    |
    | Staging S3 Access Key ID          | *from Databricks setup*    |
    | Staging S3 Secret Access Key      | *from Databricks setup*    |
    | S3 Staging Bucket Name            | *from Databricks setup*    |
    | Tasks                             | 1                          |

1. Launch the connector. Once provisioned correctly, it will write data to a Delta Lake Table automatically. Create the following table in Databricks. 
    ```sql
        CREATE TABLE orders_enriched (order_id STRING, 
            product_id STRING, size STRING, product STRING, department STRING, price STRING,
            id STRING, first_name STRING, last_name STRING, email STRING, phone STRING,
            street_address STRING, state STRING, zip_code STRING, country STRING, country_code STRING,
            partition INT) 
        USING DELTA;
    ```

1. Et voila! Now query yours records
    ```sql 
     SELECT * FROM default.orders_enriched;
    ```

Experiment to your heart's desire with the data in Databricks. For example, you could write some queries that combine the data from two tables each source database, such as caclulating total revenue by state.

</details>
<br>

<details>
    <summary><b>Snowflake</b></summary>
    
1. Follow the [source documentation](https://docs.confluent.io/cloud/current/connectors/cc-snowflake-sink.html) for full details if you wish. 
    
1. Create a private/public key pair for authenticating to your Snowflake account. 
    - In a directory outside of your repo, run the following:
    ```
    $ openssl genrsa -out snowflake_key.pem 2048
    $ openssl rsa -in snowflake_key.pem  -pubout -out snowflake_key.pub
    $ export SF_PUB_KEY=`cat snowflake_key.pub | grep -v PUBLIC | tr -d '\r\n'`
    $ export SF_PVT_KEY=`cat snowflake_key.pem | grep -v PUBLIC | tr -d '\r\n'`
    ```
    - Copy the values of each parameter into your `env.sh` file for easy access
    
1. Create a Snowflake user with permissions. Refer to the [source doc](https://docs.confluent.io/cloud/current/connectors/cc-snowflake-sink.html#creating-a-user-and-adding-the-public-key) if you need screenshots for guidance.
    - Login to your Snowflake account and select `Worksheets` from the menu bar.
    - In the upper-corner *of the Worksheet view*, set your role to `SECURITYADMIN`
    - The following steps configure the role `kafka_connector` with full permissions on database `RTDW`:
    ```
    use role securityadmin;
    create user confluent RSA_PUBLIC_KEY=<*SF_PUB_KEY*>
    create role kafka_connector;
    // Grant database and schema privileges:
    grant usage on database RTDW to role kafka_connector;
    grant usage on schema RTDW.PUBLIC to role kafka_connector;
    grant create table on schema RTDW.PUBLIC to role kafka_connector;
    grant create stage on schema RTDW.PUBLIC to role kafka_connector;
    grant create pipe on schema RTDW.PUBLIC to role kafka_connector;

    // Grant this role to the `confluent` user and make it the user's default:
    grant role kafka_connector to user confluent;
    alter user confluent set default_role=kafka_connector;    
    ```
1. To review the grants, enter:
    ```
    show grants to role kafka_connector; 
    ```    
    
1. Configure the SnowflakeSink connector
    - Review the [connector's limitations](https://docs.confluent.io/cloud/current/connectors/cc-snowflake-sink.html#quick-start)
    - Fill in the values using the following table:
    
    | **Property**                      | **Value**                  |
    |-----------------------------------|----------------------------|
    | Topic to read                     | `orders_enriched`          |
    | Input Kafka record value format   | `JSON_SR`                  |
    | Input Kafka record hey format     | `JSON`                     |
    | Kafka cluster authentication mode | `KAFKA_API_KEY`            |
    | Kafka API Key                     | from `env.sh`              |
    | Kafka API Secret                  | from `env.sh`              |
    | Connection URL                    | for Snowflake account      |
    | Connection user name              | `confluent`                |
    | Private key                       | `SF_PVT_KEY` in `env.sh`.  |
    | Database name                     | `RTDW`                     |
    | Schema name                       | `PUBLIC`                   |
    | Topic to tables mapping           | `orders_enriched:orders`   |
    | Tasks                             | `1`                        |

1. Once provisioning succeeds, the connector reads the `orders_enriched` topic, creates the Snowflake table `orders`, and starts populating it immediately. It may however take a few minutes for Snowflake to read the records from object storage and create the table.

1. Run the following commands to make your warehouse active and assume the appropriate role. You will then see a few records returned in JSON format.
    ```sql
    use warehouse <replace>;
    use role kafka_connector;
    SELECT record_content FROM rtdw.public.orders limit 100;
    ```

 1. You can flatten data in Snowflake if you wish. Use [Snowflake's documentation](https://docs.snowflake.com/en/user-guide/json-basics-tutorial-query.html). You can also query JSON data directly in Snowflake by naming the column and specifying columns of interest. For example:
    ````sql
    SELECT RECORD_CONTENT:email from rtdw.public.orders limit 100;
    
</details>

<br>

***

## Cleanup

**Delete everything you provisioned** in this lab to avoid further usage charges. If you want to keep your work but minimize uptime cost, pause your connectors.

### Confluent Cloud resources to remove
- Ksql Cluster
- Delta Lake Sink Connector
- Postgres CDC Source Connector
- Mysql CDC Source Connector
- Kafka Cluster

### Terraform
Use `terraform destroy` to clear out your cloud resources

### Databricks and Snowflake
If you created instances of either Databricks and Snowflake solely to run this lab, you can remove them.

***

## Useful Links

Databricks
- [Confluent Cloud Databricks Delta Lake Sink](https://docs.confluent.io/cloud/current/connectors/cc-databricks-delta-lake-sink/cc-databricks-delta-lake-sink.html)
- [Databricks Setup on AWS](https://docs.confluent.io/cloud/current/connectors/cc-databricks-delta-lake-sink/databricks-aws-setup.html)
