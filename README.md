# Data Pipeline Workshop

This workshop will provide good exposure to the tools and processes needed to create a complete data stream from source to target. The stream you will create is a simulation of a stock trade data stream.
We will use Astra Streaming topics/functions/sinks, the Pulsar CLI, file data sources, Astra DB with CDC, and finally ElasticSearch.

The process you will follow for this workshop is as follows:
    
* Incoming data will come from a file that will be consumed by a Pulsar source (running locally on your laptop or on GitPod).  This will provide experience in using the Pulsar CLI to interact with Astra.  
* We will deploy a function that will enrich the messages, and store them in Astra DB using a sink.
* CDC will detect changes in the Astra DB table and publish them to a data topic in Astra Streaming.
* We will deploy a function that filters messages and publishes them to multiple topics based on their content.
* Messages from those topics will be consumed with sinks to ElasticSearch.

## Prerequisites
To execute this workshop, ensure that you have the following:
* Java 11 installed
* [Pulsar 2.10.x](https://github.com/datastax/pulsar/releases/download/ls210_4.0/lunastreaming-2.10.4.0-bin.tar.gz)
* An [Astra](https://astra.datastax.com) account
* An [ElasticSearch](https://www.elastic.co/) account

### DataStax Astra
If you do not have an Astra account, create a free trial account at [Astra Registration](https://astra.datastax.com/register).

<img width="600" src="assets/astra_registration.jpg">

### ElasticSearch
If you do not have an ElastSearch account, create a free trial account and then:

* create a deployment in the uscentral1 GCP region.  <b>Be sure to save the credentials that are provided</b> as you'll need them later.
* click on your deployment in the menu on the left.
* Take note of the following items as you'll need them in later steps:
    * ElasticSearch endpoint
    * Kibana endpoint

<img width="800" src="assets/elastic_items.png">

## Setup
In this section, we will walk through setup of the tools needed to execute this workshop.  This includes the Pulsar CLI, our Astra DB table, and ElasticSearch.

### Pulsar CLI tools
1. Download the [Pulsar 2.10.x](https://github.com/datastax/pulsar/releases/download/ls210_4.0/lunastreaming-2.10.4.0-bin.tar.gz) archive. 
2. Extract the archive.
    ```sh
    tar zxf lunastreaming-2.10.4.0-bin.tar.gz
    ```
3. Make a note of this directory location as `<YOUR PULSAR DIR>`
4. Create the connectors directory:
    ```sh
    cd lunastreaming-2.10.4.0
    mkdir connectors
    ```

### Astra Streaming Tenant and Namespace
1. Log into your Astra account.

2. Create a Streaming Tenant
    1. Navigate to *Streaming* in the Menu.
    2. Click the *Create Tenant* button.
    3. Create a streaming tenant using the following:
        * Tenant Name: irt
        * Provider: Google Cloud
        * Region: useast1
    
    <img width="344" src="assets/create_streaming.png">

3. Create a Namespace & Topic
    1. Create Namespace 
        1. Navigate to *Streaming* in the Menu.
        2. Open the `irt` tenant.
        3. Navigate to the *Namespace and Topics* tab.
        4. Click the *Create Namespace* button.
        5. Enter `stocks` as the Namespace Name.
    2. Create Topic
        1. Click the *Add Topic* button in the `stocks` Namespace.
        2. Enter `stocks-in` as the Topic Name.

        <img width="800" src="assets/add_topic.png">
    3. Create Schema
        - Open the `stocks-in` Topic.
        - Navigate to the *Schema* tab.
        - Click the *Create Schema* button.
        - Select `String` as the Schema Type.

        <img width="800" src="assets/add_schema.png">

4. Add Streaming Configuration to Pulsar CLI
    1. Download configuration file
        1. Navigate to the *Connect* tab.
        2. Click the *Download client.conf* button.
    2. Add *client.conf* to Pulsar CLI
        1. Navigate to `<YOUR PULSAR DIR>/conf` on your laptop.
            ```sh
            cd <YOUR PULSAR DIR>/conf
            ```
        2. Move the existing `client.conf` file to `client.conf.local` and copy the file you just downloaded into this directory with the name `client.conf`.  
            ```sh
            mv client.conf client.conf.local
            cp <PATH TO NEW FILE>/client.conf .
            ```

## Creating the Pulsar Stream

### Pulsar from the Command Line

The various Pulsar CLIs are installed as part of the Pulsar distribution and can be found the `<YOUR PULSAR DIR>/bin` directory.

1. Change directory into the bin directory.
    ```sh
        cd <YOUR PULSAR DIR>/bin
    ```

2. Create a **consumer** using the following command.  The `-s` option specifies your subscription name and the `-n 0` option tells the client to consume messages continuously.
    ```sh
        ./pulsar-client consume -s test -n 0 irt/stocks/stocks-in
    ```

3. In a new terminal window, create a **producer** using the following command.
    ```sh
        ./pulsar-client produce -m hello irt/stocks/stocks-in
    ```
     
You should see the message content `key:[null], properties:[], content:hello` as the output in the consumer's console.  At this point you have your first topic created and you have verified that you can connect to Astra Streaming and produce/consume messages.
    
### File Source

Now that you have a topic that you can publish to, create a Pulsar file source connector running locally on your laptop and let it process an input file. You will specify a folder in which the connector will look for files.  

1. Create a directory `/tmp/stocks`
    ```sh
        mkdir /tmp/stocks
    ```

2. Install the File Connector
    1. Download the [file connector](https://www.apache.org/dyn/mirrors/mirrors.cgi?action=download&filename=pulsar/pulsar-2.10.3/connectors/pulsar-io-file-2.10.3.nar) from the Apache Pulsar site.
        ```sh
        cd <YOUR PULSAR DIR>/connectors
        wget -O pulsar-io-file-2.10.3.nar "https://www.apache.org/dyn/mirrors/mirrors.cgi?action=download&filename=pulsar/pulsar-2.10.3/connectors/pulsar-io-file-2.10.3.nar"
        ```
    2. Place this file in `<YOUR PULSAR DIR>/connectors`.

3. Collect the following information:
    * \<TOKEN\>: **authParams** in your `client.conf` file.
    * \<BROKER SERVICE URL\>: **brokerServiceUrl** in your `client.conf` file.
    * \<YOUR GITHUB PROJECT DIR\>: The location of the GitHub project folder.


4. Start the File Connector

    (The directory locations need to be the fully qualified directory and not the relative path.)
    ```sh
    ./pulsar-admin sources localrun \
        --broker-service-url <BROKER SERVICE URL> \
        --client-auth-plugin org.apache.pulsar.client.impl.auth.AuthenticationToken \
        --client-auth-params "<TOKEN>" \
        --archive <YOUR PULSAR DIR>/connectors/pulsar-io-file-2.10.3.nar \
        --name stocks-file-source \
        --destination-topic-name irt/stocks/stocks-in \
        --source-config-file <YOUR GITHUB PROJECT DIR>/stock-price-file-connector.yaml
    ```

5. Trigger a file read
    1. Start the same consumer from the previous step (if it's not still running)
    2. Place Data File
        ```sh
        cp <YOUR GITHUB PROJECT DIR>/stock-prices-10.csv /tmp/stocks
        ```

    You should see the log for the file source display processing statements, and you should see new messages output by the consumer.  There will be a message for each line in the file.


### Enrichment Function

Next we will add a function to the stream.  This function will consume messages from the `stocks-in` topic, convert the message to a Java object, add a GUID, and then publish a message as a JSON schema.  You can find the function code in `<YOUR GITHUB PROJECT DIR>src/main/java/com/datastax/demo/function/AddUuidToStockFunction.java` in the GitHub repo you cloned.  

1. Create a topic called `stocks-enriched` in your stocks namespace.

2. Compile the function
    ```sh
    cd <YOUR GITHUB PROJECT DIR>
    ./mvnw clean install
    ```
    This will create a jar file in the `<YOUR GITHUB PROJECT>/target`

3. Create an Astra Streaming Function
    1. Navigate to the `Functions` tab of the `irt` streaming tenant.
    2. Click the *Create Function* button.
    3. Create the function using:
        - Name Your Function
            - Function Name: `enrichfunction`
            - Namespace: `stocks`
        - Upload Your Code
            - Upload my own code
            - Select the file `function-demo-0.0.1-SNAPSHOT.jar`
            - Choose a function: `AddUuidToStockFunction`
        - Choose Input Topics
            - Namespace: `stocks`
            - Input Topic: `stocks-in`
        - Choose Destination Topics
            - Namespace: `stocks`
            - Output Topic: `stocks-enriched`
        - Leave the advanced configuration items set to the defaults.
    4. You can watch the startup of your function by clicking the name and scrolling to the bottom where the logs are displayed.

4. Create a **consumer** for `stocks-enriched`
    1. In a new terminal window:
    ```sh
    cd <YOUR PULSAR DIR>
    ./pulsar-client consume -s test -n 0 irt/stocks/stocks-enriched
    ```
5. Trigger a file read
    1. Place Data File
        ```sh
        cp <YOUR GITHUB PROJECT DIR>/stock-prices-10.csv /tmp/stocks
        ```

You should see messages consumed by the Pulsar client consumer we just created.  They should be in JSON format.


### Storing Data in Astra DB

The messages that are created by consuming the stock file and then enriched by the first function will be inserted into a table in Astra DB. 

1. Log into your Astra account.

2. Create a Database
    1. Navigate to *Databases* in the Menu.
    2. Click the *Create Database* button.
    3. Create the database using the following:
        * Database Name: `irt`
        * Keyspace Name: `stocks`
        * Provider: `Google`
        * Region: `us-east1`
        
    <img width="350" src="assets/create_database.png">

3. Generate a Token
    1. Click the *Generate Token* button.
    2. Click on *Download Token Details*.
    3. Open the downloaded file `irt-token.json` and verify that you can read the file.

4. Create the `stocks` Table
    1. Navigate to the `CQL Console` tab
    2. Paste the following CQL command into the CQL Console:
        ```sql
        create table stocks.stocks ( 
            uid uuid primary key,
            symbol text, 
            trade_date text, 
            open_price float, 
            high_price float, 
            low_price float, 
            close_price float, 
            volume int 
        );
        ```
    3. DO NOT enable CDC on this table yet.

5. Create a Sink
    1. Navigate to *Streaming* in the Menu.
    2. Open the `irt` Tenant.
    3. Navigate to the *Sinks* tab
    4. Click the *Create Sink* button.
    5. Create the sink using the following:
        - The Basics
            - Namespace: `stocks`
            - Sink Type: `Astra DB`
            - Name: `stocks2astradb`
        - Connect Topics
            - Input Topics: `stocks-enriched`
        - Sink-Specific Configuration
            - Database: `irt`
            - Token: `<YOUR ASTRA DB TOKEN>`
            - Keyspace: `stocks`
            - Table Name: `stocks`
            - Mapping: `uid=value.uuid,symbol=value.symbol,trade_date=value.date,open_price=value.openPrice,high_price=value.highPrice,low_price=value.lowPrice,close_price=value.closePrice,volume=value.volume`
    
        <img width="800" src="assets/create_sink.png">

6. Trigger a file read
    1. Place Data File
        ```sh
        cp <YOUR GITHUB PROJECT DIR>/stock-prices-10.csv /tmp/stocks
        ```
7. Validate the data flow into the `stocks` table
    1. Navigate to *Databases* in the Menu.
    2. Open the `irt` Database.
    3. Navigate to the `CQL Console` tab
    4. Paste the following CQL query into the CQL Console:
        ```sql
        select * from stocks.stocks;
        ```
    <img width="800" src="assets/select_stocks.png">


### Change Data Capture 

Now that we have a table holding enriched stock data, let's enable CDC and look at what gets created.
1. Enable CDC
    1. Navigate to *Databases* in the Menu.
    2. Open the `irt` Database.
    3. Navigate to the `CDC` tab
    4. Click the `Enable CDC` button
    5. Enable CDC using the following:
        - Tenants: `pulsar-gcp-east1 / irt`
        - Keyspace: `stocks`
        - Table Name: `stocks`
    
    <img width="350" src="assets/enable_cdc.png">

2. Get the CDC Data topic name
    1. Navigate to *Streaming* in the Menu.
    2. Open the `irt` Tenant.
    3. Navigate to the `Namespaces and Topics` tab.
        - notice that there is a new Namespace created by enabling CDC: `astracdc`.
    4. Expand the `astracdc` Namespace.
        - You should see a data topic and a log topic.
    5. Copy the name of the data topic for the next step.    

3. Create a **consumer** for the CDC Data topic
    1. In a new terminal window:
    ```sh
    cd <YOUR PULSAR DIR>
    ./pulsar-client consume -s test -n 0 irt/astracdc/data-xxxxxxxxxxxxxxxxx.stocks
    ```
4. Trigger a file read
    1. Place Data File
        ```sh
        cp <YOUR GITHUB PROJECT DIR>/stock-prices-10.csv /tmp/stocks
        ```

    Your consumer should receive 10 messages. 


### Stock Routing Function
The last part of the stream prior to sending data to external systems is to create the routing function in Astra Streaming.  This function will consume data from the CDC data topic and publish a new message to a topic that corresponds to the symbol in the message.

1. Create stock specific Topics
    1. Navigate to *Streaming* in the Menu.
    2. Open the `irt` tenant.
    3. Navigate to the *Namespace and Topics* tab.
    4. Click the *Add Topic* button in the `stocks` Namespace.
        1. Enter `stocks-default` as the Topic Name.
    5. Repeat for each of:
        - `stocks-aapl`
        - `stocks-goog`

2. Routing Function
    
    Look at the routing function code in your GitHub project in `src/main/java/com/datastax/demo/function/FilterStockByTicker.java`

    This code provides an example of how you can publish messages to multiple topics from one function.  It works by looking at the stock symbol field of the incoming message and filters based on the value.  It will pass all messages that match AAPL to the "stocks-aapl" topic and all messages that match "GOOG" to the "stocks-goog" topic.  All messages will be published to the `stocks-default` topic.

3. Compile the class
    ```sh
    cd <YOUR GITHUB PROJECT DIR>
    ./mvnw clean package
    ```
4. Create an Astra Streaming Function from the compiled class
    1. Navigate to the `Functions` tab of the `irt` streaming tenant.
    2. Click the *Create Function* button.
    3. Create the function using:
        - Name Your Function
            - Function Name: `routingfunction`
            - Namespace: `stocks`
        - Upload Your Code
            - Upload my own code
            - Select the file `function-demo-0.0.1-SNAPSHOT.jar`
            - Choose a function: `FilterStockByTicker`
        - Choose Input Topics
            - Namespace: `astracdc`
            - Input Topic: `data-xxxxxxxxxxxxxxxxx.stocks`
        - Choose Destination Topics
            - Namespace: `stocks`
            - Output Topic: `stocks-default`
        - Leave the advanced configuration items set to the defaults.
    4. You can watch the startup of your function by clicking the name and scrolling to the bottom where the logs are displayed.

5. Create a **consumer** for `stocks-aapl`
    1. In a new terminal window:
    ```sh
    cd <YOUR PULSAR DIR>
    ./pulsar-client consume -s test -n 0 irt/stocks/stocks-aapl
    ```
4. Trigger a file read
    1. Place Data File
        ```sh
        cp <YOUR GITHUB PROJECT DIR>/stock-prices-10.csv /tmp/stocks
        ```


### Send Data to External Targets

#### ElasticSearch Sink
The ElasticSearch sink is a built in connector for Astra Streaming.  From the setup step where you created an ElasticSearch account, you'll need the following values in order to create the sink.

- Elastic Endpoint URL
- Username / Password

1. Create a Sink to Elasticsearch
    1. Navigate to *Streaming* in the Menu.
    2. Open the `irt` Tenant.
    3. Navigate to the *Sinks* tab
    4. Click the *Create Sink* button.
    5. Create the sink using the following:
        - The Basics
            - Namespace: `stocks`
            - Sink Type: `Elastic Search`
            - Name: `stocks2elastic`
        - Connect Topics
            - Input Topics: `stocks-aapl`
        - Sink-Specific Configuration
            - Elastic Search URL: `<Elastic Endpoint URL>`
            - Index Name: `appl-index`
            - Username: `<Elastic Username>`
            - Password: `<Elastic Password>`
                - You can skip the token and API key fields
            - Ignore Record Key: Disabled
            - Strip Nulls: Disabled
            - Enable Schemas: Disabled
            - Copy Key Fields: Enabled
            - Mapping: `uid=value.uuid,symbol=value.symbol,trade_date=value.date,open_price=value.openPrice,high_price=value.highPrice,low_price=value.lowPrice,close_price=value.closePrice,volume=value.volume`
            
        - For all other values, you can leave them set to the defaults.

        <img width="800" src="assets/create_sink_elastic.png">

    6. Click the sink name on the following page and you can see the configuration and logs for the sink as it's being created.

2. Trigger a file read
    1. Once the sink is running, place Data File
        ```sh
        cp <YOUR GITHUB PROJECT DIR>/stock-prices-10.csv /tmp/stocks
        ```

3. Validate the data flow into Elasticsearch
    1. Log in to [Elasticsearch](https://cloud.elastic.co/)
    2. Open your Elasticsearch Service Deployment
    3. Click on `Enterprise Search`.  
    4. Click `Indices` in the menu.
    5. Open the index called `appl-index`.
    5. Click on the `Documents` tab.
    
    You will see records that were sent through the AAPL topic by the routing function created in the previous step.
