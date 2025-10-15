# Real-Time Anomaly Detection using KsqlDB and Kafka Connectors

This guide provides the complete, sequential steps for setting up the anomaly detection architecture in Confluent Cloud using ksqlDB and Managed Connectors.
# Phase 1: Confluent Cloud Preparation

These steps ensure your cluster environment is ready.
1. Set Up Cluster and API Keys

    Ensure you have an active Kafka Cluster (Basic or Standard is sufficient).

    Create an API Key and Secret pair for your Kafka cluster and keep them safe.

2. Create Required Topics

Create the following two topics manually in your Confluent Cloud cluster:

Topic Name
	
software_logs (The raw input for logs (produced by the Source Connector).)

anomaly_alerts (The filtered output of the ksqlDB rule (consumed by the Sink Connector).)
	

# Phase 2: Deploy ksqlDB Processing Stream (The Intelligence)

Execute these three commands sequentially in your ksqlDB Editor on Confluent Cloud.
# A. Define the Input Stream

This tells ksqlDB how to read the JSON logs from the software_logs topic.
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
```
# B. Create the Real-Time Error Counting Table

This stateful logic defines the 5-second Tumbling Window aggregation for 'ERROR' messages per service.
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
# C. Create the Anomaly Alert Output Stream

This is the final detection rule: filtering the counts to only alert when the count is greater than 5.
```
CREATE STREAM anomaly_alerts WITH (KAFKA_TOPIC='anomaly_alerts') AS
  SELECT
    service,
    error_count,
    'CRITICAL: High Error Rate Detected' AS alert_message,
    TIMESTAMPTOSTRING(WINDOWSTART, 'yyyy-MM-dd HH:mm:ss Z') AS window_start
  FROM error_counts
  WHERE error_count > 5
  EMIT CHANGES;
```
# Phase 3: Configure the Source Connector (Ingestion)

This step simulates the log producer by reading from a file and writing into Kafka.
1. Prepare Local Log File

Create a file named app_simulation.log on your machine where the connector will run:

```
{"timestamp":"2025-10-15T12:00:00","service":"auth_service","level":"INFO","message":"User login successful"}
{"timestamp":"2025-10-15T12:00:01","service":"billing_api","level":"INFO","message":"Request completed in 120ms"}
````
2. Deploy the FileStreamSource Connector

Use the Confluent Cloud Connector UI or API to deploy this configuration. Replace [PATH_TO_FILE] with the absolute path to your app_simulation.log.

```
{
  "name": "LogFileSourceConnector",
  "config": {
    "connector.class": "FileStreamSource",
    "tasks.max": "1",
    "topic": "software_logs",
    "file": "[PATH_TO_FILE]/app_simulation.log",
    
    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter.schemas.enable": "false",
    
    "errors.tolerance": "all",
    "errors.log.enable": "true",
    "name": "LogFileSourceConnector"
  }
}
```
# Phase 4: Configure the Sink Connector (Alert Delivery)

This step automates alert delivery to an external system (e.g., PagerDuty, Slack via WebHook).
# 1. Obtain WebHook URL

Get the API endpoint (WebHook URL) for the system where you want the alerts delivered.
# 2. Deploy the HTTP Sink Connector

Use the Confluent Cloud Connector UI or API. Replace [YOUR_ALERT_WEBHOOK_URL] with the actual destination URL.
```
{
  "name": "AnomalyAlertHTTPSink",
  "config": {
    "connector.class": "io.confluent.connect.http.HttpSinkConnector",
    "tasks.max": "1",
    "topics": "anomaly_alerts",
    
    "http.api.url": "[YOUR_ALERT_WEBHOOK_URL]",
    "request.method": "POST",
    
    "headers": "Content-Type: application/json",
    
    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter.schemas.enable": "false",
    
    "reporter.result.topic.name": "http_sink_results",
    "reporter.result.topic.replication.factor": "1",
    "reporter.result.topic.partitions": "1",
    "name": "AnomalyAlertHTTPSink"
  }
}
```
# Phase 5: Test the Pipeline (Trigger Anomaly)

Start both the Source Connector and the Sink Connector.

Manually append 6 or more ERROR lines for the same service (e.g., billing_api) to your app_simulation.log file within a short time frame (less than 5 seconds).

Example lines to append:
```
    {"timestamp":"2025-10-15T12:00:10","service":"billing_api","level":"ERROR","message":"DB connection failed"}
    {"timestamp":"2025-10-15T12:00:10","service":"billing_api","level":"ERROR","message":"DB connection failed"}
    {"timestamp":"2025-10-15T12:00:11","service":"billing_api","level":"ERROR","message":"DB connection failed"}
    {"timestamp":"2025-10-15T12:00:11","service":"billing_api","level":"ERROR","message":"DB connection failed"}
    {"timestamp":"2025-10-15T12:00:12","service":"billing_api","level":"ERROR","message":"DB connection failed"}
    {"timestamp":"2025-10-15T12:00:12","service":"billing_api","level":"ERROR","message":"DB connection failed"}
```
The HTTP Sink Connector will automatically fire the alert to your WebHook within seconds!



self-contained HTML application can be accessed using index.html file  that simulates the entire end-to-end flow described.
Download the [index.html](https://github.com/varsharma-12/log-anamoly-detecter/blob/main/index.html) and run it in a browser to see a simulated environment.
