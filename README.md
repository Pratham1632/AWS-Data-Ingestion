# AWS Data Ingestion Project

This project demonstrates how to build a complete end-to-end data ingestion and analytics pipeline using **Amazon Web Services (AWS)**. The pipeline fetches real-time stock data from a public API, ingests it using **AWS Lambda**, routes it through **Kinesis Data Firehose**, stores it in **S3**, processes it with **AWS Glue**, queries it with **Amazon Athena**, and visualizes the insights in **Power BI**.

## 📊 Use Case

- Real-time stock data analytics (example: IBM)
- Scalable, serverless data ingestion
- Cloud-native ETL and BI pipeline

## 🧰 Tools & Services Used

- AWS Lambda
- Amazon Kinesis Data Firehose
- Amazon S3
- AWS Glue
- Amazon Athena
- Power BI

## 📈 Data Source

- [Alpha Vantage - IBM Intraday Stock API (5min interval)](https://www.alphavantage.co/query?function=TIME_SERIES_INTRADAY&symbol=IBM&interval=5min&apikey=demo)

---

## 🔧 Step-by-Step Implementation

### 1. Set Up AWS Account

Follow the [official guide](https://docs.aws.amazon.com/accounts/latest/reference/manage-acct-creating.html) to create an AWS account.

### 2. Create an S3 Bucket

- Disable public access
- Leave "Object Lock" disabled

### 3. Create a Kinesis Firehose Delivery Stream

- **Source**: Direct PUT
- **Destination**: S3 bucket created earlier
- **Buffer size**: 5 MiB
- **Buffer interval**: 60 seconds
- **IAM Role**: Attach `AmazonS3FullAccess`, `AWSLambdaFullAccess`

### 4. Create Lambda Function for Data Ingestion

```python
import boto3
import json
import urllib3

def lambda_handler(event, context):
    http = urllib3.PoolManager()
    
    response = http.request(
        "GET", 
        "https://www.alphavantage.co/query?function=TIME_SERIES_INTRADAY&symbol=IBM&interval=5min&apikey=demo"
    )
    
    response_dict = json.loads(response.data.decode("utf-8"))
    time_series = response_dict.get("Time Series (5min)")
    
    if time_series:
        firehose = boto3.client('firehose')
        for timestamp, data in time_series.items():
            record = {
                "time": timestamp,
                "open": data.get("1. open"),
                "high": data.get("2. high"),
                "low": data.get("3. low"),
                "close": data.get("4. close"),
                "volume": data.get("5. volume")
            }
            firehose.put_record(
                DeliveryStreamName="PUT-S3-YourStreamNameHere",
                Record={'Data': json.dumps(record) + '\n'}
            )


### 5. Configure Amazon Athena

* Create an S3 bucket to store query results
* Set the output location in Athena settings
* Create database:

  ```sql
  CREATE DATABASE stock_db;
  ```

### 6. Use AWS Glue Crawler

* Source: S3 (from Firehose)
* Target: Athena database
* IAM Role: Grant access to S3 and Athena
* Table name prefix: `stock_data_`

### 7. Query Ingested Data in Athena

```sql
SELECT * FROM stock_db.stock_data_fh LIMIT 100;
```

### 8. Create ETL Table Using Athena Query (via Glue Job)

```python
import boto3

client = boto3.client('athena')

client.start_query_execution(
    QueryString="""
        CREATE TABLE stock_db.stock_analysis
        WITH (
            external_location = 's3://your-parquet-output-bucket/',
            format = 'PARQUET',
            write_compression = 'SNAPPY',
            partitioned_by = ARRAY['time']
        )
        AS
        SELECT open, high, low, time
        FROM stock_db.stock_data_fh;
    """,
    QueryExecutionContext={'Database': 'stock_db'},
    ResultConfiguration={'OutputLocation': 's3://your-athena-results-bucket/'}
)
```

### 9. Visualize in Power BI

* Use the **Athena ODBC** or **Amazon Athena connector**
* Refer to:

  * [YouTube Tutorial 1](https://www.youtube.com/watch?v=ClBQ3_p7T_A)
  * [YouTube Tutorial 2](https://www.youtube.com/watch?v=FKdCr6vmq-o&t=326s)

---

## ✅ Outcome

Successfully implemented a serverless, cloud-native data pipeline that ingests, stores, transforms, and visualizes stock data using AWS and Power BI. This project showcases how modern cloud services can be combined for scalable and real-time analytics.

---

## 📁 Directory Structure (Example)

```
aws-data-ingestion/
├── lambda/
│   └── lambda_function.py
├── glue/
│   └── etl_query.py
├── screenshots/
│   └── powerbi_dashboard.png
├── README.md
```

---

## 🚀 Future Enhancements

* Add alerts using Amazon SNS
* Schedule Lambda using EventBridge
* Integrate more stock tickers and data sources
* Implement real-time dashboards using Amazon QuickSight


```
