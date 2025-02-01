# Snowflake-Snowpipe-Project
🎵 AWS-Based ETL Pipeline for Spotify Data → Snowflake

### Overview
This project involves building an ETL (Extract, Transform, Load) data pipeline that extracts product data from the Flipkart API, transforms it into the desired format, and loads it into a Snowflake database using Auto-Ingest Snowpipe. This enables real-time or near real-time data ingestion, ensuring efficient and automated data processing for further analysis and reporting.

### Architecture
![Architecture Diagram](https://github.com/sahil118/flipkart-end-to-end-data-engineering-project/blob/main/flipkart-project-architecture.png)

### About API/Dataset:
The Real-Time Flipkart API provides product details such as name, price, and ratings based on their popularity. [API Endpoint](https://rapidapi.com/opendatapoint-opendatapoint-default/api/real-time-flipkart-api/playground).

### Tools Utilized

1. **S3(Amazon Simple Storage Service):** Amazon S3, a scalable object storage service provided by Amazon Web Services (AWS). Buckets are used to store and organize data, such as documents, images, backups, logs, and other types of files.

2. **AWS Lambda :** AWS Lambda is a serverless compute service provided by Amazon Web Services (AWS) that allows you to run code without provisioning or managing servers. With AWS Lambda, you can upload your code, set triggers, and the service automatically executes the code in response to specific events, such as changes to data in an S3 bucket, updates in a DynamoDB table, or HTTP requests via Amazon API Gateway.

3. **Snowflake database** : A Snowflake database is a cloud-based, fully managed data warehouse that provides high-performance storage, processing, and analytics for structured and semi-structured data. It is built on a multi-cluster shared data architecture, enabling scalability, flexibility, and ease of use without the need for traditional database management tasks.

4. **Snowpipe :** Snowpipe is Snowflake’s continuous data ingestion service that allows users to automatically load streaming or batch data into Snowflake as soon as it becomes available in a cloud storage location (e.g., AWS S3, Azure Blob, Google Cloud Storage).

### Code for extraction Raw data into s3 : 
```
import http.client
import json
import os
import boto3
from datetime import datetime


def lambda_handler(event, context):
    api_key = os.environ.get('RAPIDAPI_KEY')  
    api_host = os.environ.get('RAPIDAPI_HOST')
    conn = http.client.HTTPSConnection("real-time-flipkart-api.p.rapidapi.com")

    headers = {
        'x-rapidapi-key': api_key,
        'x-rapidapi-host': api_host
    }

    conn.request("GET", "/products-by-brand?brand_id=tyy%2C4io&page=1&sort_by=popularity", headers=headers)

    res = conn.getresponse()
    data = res.read()

    decoded_data = data.decode("utf-8")
    parsed_data = json.loads(decoded_data) 

    client = boto3.client('s3')
    filename = "flipkart_raw_" + str(datetime.now()) +  ".json" #loding it into s3 bucket (Raw data)
    client.put_object(
        Bucket = "flipkart-data-project-sahil",
        Key="flip-raw-data/pending-raw-data/" + filename,
        Body = json.dumps(parsed_data)
    )
   

```
### Code for transformation Raw data into structured : 
```
import json
import boto3
from datetime import datetime
from io import StringIO
import pandas as pd 

def product(data):
    product_list = []
    for row in data['products']:
        product_id = row['pid']
        Brand_Name = row['brand']
        Product_name = row['title']
        link = row['url']
        badge = row['badge']
        Mrp_price = row['mrp']
        Actuall_price = row['price']
        Product_fetures = row['highlights']
        product_element = {'product_id':product_id,'Brand_Name':Brand_Name,'Product_name':Product_name,
                        'link':link,'badge':badge,'Mrp_price':Mrp_price,'Selling_price':Actuall_price,'Fetures':Product_fetures}
        product_list.append(product_element)
    
    return product_list

def rating(data):
    
    rating_list = []
    for row in data['products']:
        product_id = row['pid']
        Average_review = row['rating']['average']
        total_ratings = row['rating']['count']
        Review_count = row['rating']['reviewCount']
        # Breaking down the 'breakup' list into individual star ratings
        one_star = row['rating']['breakup'][0]  # Number of 1-star ratings
        two_star = row['rating']['breakup'][1]  # Number of 2-star ratings
        three_star = row['rating']['breakup'][2]  # Number of 3-star ratings
        four_star = row['rating']['breakup'][3]  # Number of 4-star ratings
        five_star = row['rating']['breakup'][4]  # Number of 5-star ratings
        rating_elements = {'Average_review':Average_review,'total_ratings':total_ratings,
                        'Feed_back':Review_count,'one_star':one_star,'two_star':two_star,
                        'three_star':three_star,'four_star':four_star,'five_star':five_star,'product_id':product_id}
        rating_list.append(rating_elements)
    
    return rating_list

def lambda_handler(event, context):
    s3 = boto3.client('s3')
    Bucket = "flipkart-data-project-sahil"
    Key = "flip-raw-data/pending-raw-data/"


    flipkart_data = []
    flipkart_keys = []
    for file in s3.list_objects(Bucket=Bucket,Prefix=Key)['Contents']:
        file_key = file['Key']
        if file_key.split('.')[-1] == "json":
            response = s3.get_object(Bucket=Bucket,Key=file_key)
            content = response['Body']
            jsonObject = json.loads(content.read())
            flipkart_data.append(jsonObject)
            flipkart_keys.append(file_key)
    

    for data in flipkart_data:
        product_list = product(data)
        rating_list =  rating(data)

        prd_df = pd.DataFrame.from_dict(product_list)
        prd_df = prd_df.drop_duplicates(subset=['product_id'])

        rating_df = pd.DataFrame.from_dict(rating_list)
        rating_df = rating_df.drop_duplicates(subset=['product_id'])

        product_key = "flip-trans-data/product-details-data/product_transformed_data" + str(datetime.now()) + ".csv"
        product_buffer = StringIO()
        prd_df.to_csv(product_buffer,index=False)
        prd_content = product_buffer.getvalue()
        s3.put_object(Bucket=Bucket, Key=product_key, Body=prd_content)

        rating_key = "flip-trans-data/rating-details-data/rating_transformed_data" + str(datetime.now()) + ".csv"
        rating_buffer = StringIO()
        rating_df.to_csv(rating_buffer,index=False)
        rating_content = rating_buffer.getvalue()
        s3.put_object(Bucket=Bucket, Key=rating_key, Body=rating_content)
    

    s3_resource = boto3.resource('s3')
    for key in flipkart_keys:
        copy_source = {
            'Bucket' : Bucket,
            'Key' : key
        }
        s3_resource.meta.client.copy(copy_source,Bucket,"flip-raw-data/processed-raw-data/" + key.split("/")[-1])
        s3_resource.Object(Bucket,key).delete()


```

### Efficiently ingests it into a Snowflake database using Snowpipe’s automatic loading, ensuring seamless real-time data processing and analytics:

### Snowflake table creation :  
```
CREATE OR REPLACE DATABASE SPOTIFY_DATA;

CREATE OR REPLACE TABLE SPOTIFY_DATA.PUBLIC.album(
    album_id varchar PRIMARY KEY ,
    album_name varchar,
    album_release_date date,
    album_total_track varchar,
    album_url varchar
);

CREATE OR REPLACE TABLE artist(
    artis_id VARCHAR PRIMARY KEY,
    artist_name varchar,
    external_url varchar
);

CREATE OR REPLACE TABLE song(
    song_id VARCHAR PRIMARY KEY,
    album_id varchar,
    artist_id varchar,
    song_name varchar,
    song_popularity varchar,
    song_duration varchar,
    song_url varchar,
    song_added_date date
    
);
```

### Snowflake storage integration creation :  
```
CREATE OR REPLACE STORAGE INTEGRATION spo_s3
    TYPE = EXTERNAL_STAGE
    STORAGE_PROVIDER = 's3'
    ENABLED = TRUE
    STORAGE_AWS_ROLE_ARN = '##'
    STORAGE_ALLOWED_LOCATIONS = ('s3://spotify-etl-project-sahil')
    comment = 'Creating connection to s3';
```

### Snowflake file format creation : 
```
CREATE OR REPLACE FILE FORMAT spotify_data.FILE_FORMAT.spotify_data_CSV
type = csv
skip_header = 1
field_delimiter = ','
empty_field_as_null = True
RECORD_DELIMITER = '\n'
FIELD_OPTIONALLY_ENCLOSED_BY = '"'
null_if = ('NULL','null');
```

### Snowflake stage creation : 
```
CREATE OR REPLACE STAGE SPOTIFY_DATA.EXTERNAL_STAGE.spo_stage
    url = 's3://spotify-etl-project-sahil/tranformed_data/'
    STORAGE_INTEGRATION = spo_s3
    file_format =spotify_data.FILE_FORMAT.spotify_data_CSV;
```

### Snowflake snowpipe creation for album_table: 
```
CREATE OR REPLACE PIPE spotify_data.pipe.spo_pipe_album 
auto_ingest = True
as 
COPY INTO SPOTIFY_DATA.PUBLIC.ALBUM
from @SPOTIFY_DATA.EXTERNAL_STAGE.spo_stage
pattern = 'album_data/.*';
```

### Snowflake snowpipe creation for artist_table:
```
CREATE OR REPLACE PIPE spotify_data.pipe.spo_p   ipe_artist 
auto_ingest = True
as 
COPY INTO SPOTIFY_DATA.PUBLIC.artist
from @SPOTIFY_DATA.EXTERNAL_STAGE.spo_stage
pattern = 'artist_data/.*';
```

### Snowflake snowpipe creation for song_table:
```

CREATE OR REPLACE PIPE spotify_data.pipe.spo_pipe_song 
auto_ingest = True
as 
COPY INTO SPOTIFY_DATA.PUBLIC.song
from @SPOTIFY_DATA.EXTERNAL_STAGE.spo_stage
pattern = 'song_data/.*';
```
### Project Implementation : 
Here is your project execution description, paraphrased and broken down step by step:

1. **Data Extraction** : AWS Lambda is used as a serverless function to extract product data from the Flipkart API at regular intervals.The extraction script is deployed to AWS Lambda, ensuring automatic execution without needing a dedicated server.AWS CloudWatch Events is configured to trigger the Lambda function every hour, ensuring continuous data retrieval.The extracted data is fetched in JSON format) and temporarily stored in Lambda’s memory.If required, error handling and logging mechanisms are integrated using AWS CloudWatch Logs for monitoring and troubleshooting.
  
2.  **Store Raw Data** : Once data is extracted, it is directly stored in an AWS S3 bucket for persistent storage.S3 acts as a centralized data lake, ensuring scalability, security, and cost-effective storage.
The raw data is stored in its original format (JSON, CSV, or Parquet) to maintain flexibility for further processing.S3 bucket policies and access control settings are applied to ensure secure data storage and access control.Versioning can be enabled in the S3 bucket to track changes and maintain historical data if needed.

3. **Data Transformation** : A transformation function is implemented within AWS Lambda to process the raw data stored in S3.The function cleans, structures, and converts the raw data into a normalized format suitable for analytics.Transformation steps may include:Filtering and cleaning: Removing missing or inconsistent records.Standardization: Converting data types and formatting fields consistently.Enrichment: Adding calculated fields or additional metadata if needed.The transformed data is then stored in another S3 bucket (or a separate folder within the same bucket) for downstream processing.An S3 event trigger can be set up to automatically execute the transformation function whenever new raw data is uploaded.

4. **Data Loading into Snowflake** : 


    A Snowflake table is created to store the transformed data from the S3 bucket. The schema is designed based on the data structure to ensure efficient querying and analysis.
    The data loading process is fully automated using Snowpipe, which enables real-time or near real-time ingestion.

    Snowpipe’s auto-ingest feature is configured to detect new files in the S3 bucket as soon as they are uploaded.
    A notification mechanism (such as AWS S3 event notifications with Amazon SNS or SQS) is set up to trigger Snowpipe whenever new transformed data is added to S3.

    Snowpipe uses a COPY INTO command to efficiently load the data from S3 into the Snowflake table, ensuring optimized performance.
    Data validation and monitoring mechanisms are implemented using Snowflake’s query history, metadata tables, and logging features to track ingestion status and troubleshoot issues if needed.

    Once loaded, the data is ready for further analysis, reporting, and integration with BI tools or machine learning models.

### Conclusion : 
This automated pipeline ensures seamless data ingestion, minimal latency, and real-time availability of Flipkart product data in Snowflake, making it highly efficient for business intelligence and analytics.
