# Real-Time Anomaly Detection using Kafka Connectors

This guide details the steps and configuration required to implement the Log Anomaly Detection architecture using Kafka Connect, replacing the custom Python Producer and Consumer scripts.

Prerequisites:

    A running Kafka Cluster (Confluent Cloud or self-managed).

    The following topics already exist: software_logs, anomaly_alerts.

    The ksqlDB processing stream from the previous step is deployed and running.

    Access to a Kafka Connect cluster or the Confluent Cloud Connect UI.

# Step 1: Configure the Source Connector (Ingestion)

This connector will watch a local file (simulating your application's log file) and continuously stream its new content into the software_logs Kafka topic.
# A. Prepare the Simulated Log File

For a real-world scenario, the source connector would attach directly to the live log output. For a local test, create a file named app_simulation.log with some initial data:

```

{"timestamp":"2025-10-15T12:00:00","service":"auth_service","level":"INFO","message":"User login successful"}
{"timestamp":"2025-10-15T12:00:01","service":"billing_api","level":"INFO","message":"Request completed in 120ms"}
```
# B. Source Connector Configuration (FileStreamSource)

Use the following configuration to set up the connector. Remember to replace [PATH_TO_FILE] with the absolute path to your app_simulation.log file.

Goal: Read app_simulation.log and write the JSON lines to the software_logs topic.

````
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
````

# Step 2: KSQL Real-Time Processing (Detection Core)

This step remains unchanged from the previous implementation. The ksqlDB logic is responsible for the stateful, real-time calculation and detection.

Logic Recap:

    Input: Stream data from the software_logs topic.

    Windowing: Calculate COUNT(*) of 'ERROR' messages within a 5-second tumbling window, grouped by service.

    Alerting: Filter the windowed counts where error_count > 5 and publish the result to the anomaly_alerts topic.

# Step 3: Configure the Sink Connector (Alert Delivery)

This connector will automatically read the confirmed anomaly events from the anomaly_alerts topic and forward them to an external system, such as a monitoring tool or a WebHook that triggers a Slack message (replacing consumer.py).
A. Sink Connector Configuration (HTTP Sink)

This example uses a generic HTTP Sink Connector to send the JSON alert payload to an external WebHook endpoint. You would replace [YOUR_ALERT_WEBHOOK_URL] with the actual URL (e.g., PagerDuty, Datadog, or a custom notification service).

Goal: Read the JSON alerts from the anomaly_alerts topic and POST them to a WebHook.

````
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
````

B. Triggering an Anomaly with the Source Connector

To trigger a test alert:

    Start both connectors.

    Manually append several ERROR lines for the same service (e.g., billing_api) to your app_simulation.log file within a short time (e.g., 5 seconds).

````
{"timestamp":"2025-10-15T12:00:10","service":"billing_api","level":"ERROR","message":"Transaction failure: lock timeout"}
{"timestamp":"2025-10-15T12:00:10","service":"billing_api","level":"ERROR","message":"Transaction failure: lock timeout"}
{"timestamp":"2025-10-15T12:00:10","service":"billing_api","level":"ERROR","message":"Transaction failure: lock timeout"}
{"timestamp":"2025-10-15T12:00:11","service":"billing_api","level":"ERROR","message":"Transaction failure: lock timeout"}
{"timestamp":"2025-10-15T12:00:12","service":"billing_api","level":"ERROR","message":"Transaction failure: lock timeout"}
{"timestamp":"2025-10-15T12:00:12","service":"billing_api","level":"ERROR","message":"Transaction failure: lock timeout"}
````

The Source Connector will detect these new lines, write them to software_logs, ksqlDB will count them, and the Sink Connector will deliver the final alert via HTTP.


self-contained HTML application can be accessed using index.html file  that simulates the entire end-to-end flow described.
