# Real-Time Log Anomaly Detector Deployment Guide

This guide details the deployment of a low-latency pipeline in Confluent Cloud that detects a sudden surge of errors in application logs using ksqlDB and Managed Connectors.

Architecture Overview
   
1. Source: Datagen Source Connector generates simulated logs (LogMessages).

2. Topic: software_logs (Raw Input).

3. Processor: ksqlDB runs a 5-second tumbling window to count errors per service.

4. Filter: If ERROR_COUNT > 5, an alert is published.

5. Alert Topic: software_logs (Final Output).

6. Sink: HTTP Sink Connector delivers the alert to a WebHook.

# Phase 1: Environment Setup (Topics & WebHook)
1. Create Topics (software_logs , anamoly_alerts)

2.  Prepare WebHook Endpoint

Go to a free testing service like webhook.site or requestbin.com and copy the unique URL. This will be the destination for  alerts.

This setup uses:
Alert Destination URL: https://webhook.site/e6c81441-80dd-4f73-a171-70592a7a1e2b
https://webhook.site/#!/view/e6c81441-80dd-4f73-a171-70592a7a1e2b/b32b3a6f-d48f-4308-a237-a25a359a608d/1

# Phase 2: Deploy ksqlDB Logic (The Intelligence)

Execute these three commands sequentially in your Confluent Cloud ksqlDB Editor. Wait for confirmation before running the next command.
1. Define the Input Stream

This defines the schema for the raw log data coming into the pipeline.

```
CREATE STREAM log_stream (
    timestamp VARCHAR,
    service VARCHAR,
    level VARCHAR,
    message VARCHAR
) WITH (
    KAFKA_TOPIC='software_logs',
    VALUE_FORMAT='JSON'
);
````

2. Create the Real-Time Error Counting Table

This creates a stateful table that performs the continuous aggregation (counting errors in 5-second windows).

```
CREATE TABLE error_counts AS
  SELECT
    service,
    COUNT(*) AS error_count
  FROM log_stream
  WINDOW TUMBLING (SIZE 5 SECONDS)
  WHERE level = 'ERROR'
  GROUP BY service
  EMIT CHANGES;
```

3. Define the Anomaly Alert Output Stream

This is the final rule: it selects data from the table only when the error count is greater than 5 and publishes the structured alert message.

````
CREATE STREAM anomaly_alerts WITH (KAFKA_TOPIC='anomaly_alerts') AS
  SELECT
    service,
    error_count,
    'CRITICAL: High Error Rate Detected' AS alert_message,
    TIMESTAMPTOSTRING(WINDOWSTART, 'yyyy-MM-dd HH:mm:ss Z') AS window_start
  FROM error_counts
  WHERE error_count > 5
  EMIT CHANGES;
````

# Phase 3: Connector Deployment (Ingestion & Delivery)

Navigate to the Connectors section in Confluent Cloud and create two connectors using the JSON configuration below.
1. Datagen Source Connector (Ingestion)
    Purpose: Generates sample log data (LogMessages schema) directly to the software_logs topic.
````
{
  "name": "AnomalyDetectorSource",
  "config": {
    "connector.class": "io.confluent.kafka.connect.datagen.DatagenConnector",
    "kafka.topic": "software_logs",
    "quickstart": "LogMessages",
    "max.interval": "100",
    "iterations": "100000",
    "tasks.max": "1",
    "output.format": "json"
  }
}
````
<img width="1648" height="858" alt="software_logs" src="https://github.com/user-attachments/assets/b21328e1-825e-463b-a4ea-8b5c316dfd99" />

2. HTTP Sink Connector (Delivery)

    Consumes alerts from anomaly_alerts and posts them to your WebHook.
```
{
  "name": "AnomalyAlertHTTPSink",
  "config": {
    "connector.class": "io.confluent.connect.http.HttpSinkConnector",
    "tasks.max": "1",
    "topics": "anomaly_alerts",
    "http.api.url": "[https://webhook.site/e6c81441-80dd-4f73-a171-70592a7a1e2b](https://webhook.site/e6c81441-80dd-4f73-a171-70592a7a1e2b)", 
    "request.method": "POST",
    "headers": "Content-Type: application/json",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter.schemas.enable": "false"
  }
}
```

# Phase 4: Validation (Confirming the Flow)

  Ensure both AnomalyDetectorSource and AnomalyAlertHTTPSink show the RUNNING status in the Connectors list.

  Navigate to your WebHook URL: https://webhook.site/e6c81441-80dd-4f73-a171-70592a7a1e2b.

  Wait 10-20 seconds for the first ksqlDB window to close.

  You should see new POST requests appear on the left side of the WebHook page, containing a JSON body confirming the critical alert.


  Stream Lineage view :

  <img width="1467" height="619" alt="image" src="https://github.com/user-attachments/assets/fba31b56-7ba6-4645-8508-fef676e6075c" />

