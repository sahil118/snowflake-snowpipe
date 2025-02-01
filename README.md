# Snowflake-Snowpipe-Project
🎵 AWS-Based ETL Pipeline for Spotify Data → Snowflake

### Overview
This project involves building an ETL (Extract, Transform, Load) data pipeline that extracts top trending songs from the Spotify API, transforms it into the desired format, and loads it into a Snowflake database using Auto-Ingest Snowpipe. This enables real-time or near real-time data ingestion, ensuring efficient and automated data processing for further analysis and reporting.

### Architecture
![Architecture Diagram](https://github.com/sahil118/snowflake-snowpipe/blob/main/Screenshot%202025-02-01%20121254.png)

### About API/Dataset:
The Real-Time Spotify API provides songs and their ablum,artists based on their popularity. [API Endpoint](https://open.spotify.com/playlist/4z5whwZPQuMotubMwwlsLB).

### Tools Utilized

1. **S3(Amazon Simple Storage Service):** Amazon S3, a scalable object storage service provided by Amazon Web Services (AWS). Buckets are used to store and organize data, such as documents, images, backups, logs, and other types of files.

2. **AWS Lambda :** AWS Lambda is a serverless compute service provided by Amazon Web Services (AWS) that allows you to run code without provisioning or managing servers. With AWS Lambda, you can upload your code, set triggers, and the service automatically executes the code in response to specific events, such as changes to data in an S3 bucket, updates in a DynamoDB table, or HTTP requests via Amazon API Gateway.

3. **Snowflake database** : A Snowflake database is a cloud-based, fully managed data warehouse that provides high-performance storage, processing, and analytics for structured and semi-structured data. It is built on a multi-cluster shared data architecture, enabling scalability, flexibility, and ease of use without the need for traditional database management tasks.

4. **Snowpipe :** Snowpipe is Snowflake’s continuous data ingestion service that allows users to automatically load streaming or batch data into Snowflake as soon as it becomes available in a cloud storage location (e.g., AWS S3, Azure Blob, Google Cloud Storage).

### Code for extraction Raw data into s3 : 
```
import json
import os
import spotipy
from spotipy.oauth2 import SpotifyClientCredentials
import boto3
from datetime import datetime
def lambda_handler(event, context):
    client_id = os.environ.get('client_id')
    client_secret = os.environ.get('client_secret')
    #cache_file = "/tmp/spotify_cache.cache"
    client_credentials_manager = SpotifyClientCredentials(client_id = client_id , client_secret=client_secret)
    sp = spotipy.Spotify(client_credentials_manager = client_credentials_manager)
    playlists = sp.user_playlists('spotify')
    #sp.cache_path = cache_file
    playlist_link = "https://open.spotify.com/playlist/4z5whwZPQuMotubMwwlsLB"
    playlist_uri = playlist_link.split("/")[-1]
    spotify_data = sp.playlist_tracks(playlist_uri)

    client = boto3.client('s3')
    filename = "spotify_raw_" + str(datetime.now()) + ".json"
    client.put_object(
        Bucket="spotify-etl-project-sahil",
        Key="raw-data/to_processed/" + filename,
        Body=json.dumps(spotify_data)
    )

   

```
### Code for transformation Raw data into structured : 
```
import json
import boto3
from io import StringIO
import pandas as pd
from datetime import datetime
def album(data):
    album_list = []
    for row in data['items']:
        album_id = row['track']['album']['id']
        album_name = row['track']['album']['name']
        album_release_date = row['track']['album']['release_date']
        album_total_track = row['track']['album']['total_tracks']
        album_url = row['track']['album']['external_urls']['spotify']
        album_element = {'album_id':album_id,'name':album_name,'release_date':album_release_date,'track':album_total_track,'url':album_url}
        album_list.append(album_element)
    return album_list

def artist(data):
    artist_list = []
    for row in data['items']:
        #print(row.items())
        for key,values in row.items():
            if key == 'track':
                for artist in values['artists']:
                    artist_dict = {'artist_id' : artist['id'],'artist_name': artist['name'],'external_url':artist['href']}
                    artist_list.append(artist_dict)
    return artist_list

def songs(data):
    song_list = []
    for row in data['items']:
        song_id = row['track']['id']
        album_id = row['track']['album']['id']
        for row2 in row['track']['album']['artists']:
            artist_id = row2['id']
        song_name = row['track']['name']
        song_popularity = row['track']['popularity']
        song_duration = row['track']['duration_ms']
        song_url = row['track']['external_urls']['spotify']
        song_added_date = row['added_at']
        song_dict={'song_id':song_id,'album_id':album_id,'artist_id':artist_id,'song_name':song_name,'song_popularity':song_popularity,
                'song_duration':song_duration,'song_url':song_url,'song_added_date':song_added_date}
        song_list.append(song_dict)
    return song_list


def lambda_handler(event, context):
    s3 = boto3.client('s3')
    Bucket = "spotify-etl-project-sahil"
    Key = "raw-data/to_processed/"

    spotify_data = []
    spotify_key = []
    for file in s3.list_objects(Bucket=Bucket,Prefix=Key)['Contents']:
        file_key = file['Key']
        if file_key.split('.')[-1] == 'json':
            response = s3.get_object(Bucket=Bucket,Key=file_key)
            content = response['Body']
            jsonObject = json.loads(content.read())
            print(jsonObject)
            spotify_data.append(jsonObject)
            spotify_key.append(file_key)

    for data in spotify_data:
        album_list = album(data)
        artist_list = artist(data)
        song_list = songs(data)
    
        album_df = pd.DataFrame.from_dict(album_list)
        album_df = album_df.drop_duplicates(['album_id'])

        artist_df = pd.DataFrame.from_dict(artist_list)
        artist_df = artist_df.drop_duplicates(['artist_id'])
        artist_df.drop(artist_df[artist_df['artist_name'] == ''].index,inplace=True)
        artist_df.reset_index(drop=True,inplace=True)

        song_df = pd.DataFrame.from_dict(song_list)
        song_df = song_df.drop_duplicates(['song_id'])
        song_df = song_df.drop_duplicates(['album_id'])
        song_df = song_df.drop_duplicates(['artist_id'])
        song_df.drop(song_df[song_df['song_name'] == ''].index,inplace=True)
        song_df.reset_index(drop=True,inplace=True)

        album_df['release_date'] = pd.to_datetime(album_df['release_date'],format='%Y-%m-%d',errors='coerce')
        album_df.dropna(subset=['release_date'], inplace=True)
        album_df.reset_index(drop=True, inplace=True)
        song_df['song_added_date'] = pd.to_datetime(song_df['song_added_date'])

        songs_key = "tranformed_data/song_data/songs_transformed_"+str(datetime.now())+".csv"
        song_buffer=StringIO()
        song_df.to_csv(song_buffer,index=False)
        song_content = song_buffer.getvalue()
        s3.put_object(Bucket=Bucket,Key=songs_key,Body=song_content)

        album_key = "tranformed_data/album_data/album_transformed_"+str(datetime.now())+".csv"
        album_buffer=StringIO()
        album_df.to_csv(album_buffer,index=False)
        album_content = album_buffer.getvalue()
        s3.put_object(Bucket=Bucket,Key=album_key,Body=album_content)

        artist_key = "tranformed_data/artist_data/artist_transformed_"+str(datetime.now())+".csv"
        artist_buffer=StringIO()
        artist_df.to_csv(artist_buffer,index=False)
        artist_content = artist_buffer.getvalue()
        s3.put_object(Bucket=Bucket,Key=artist_key,Body=artist_content)
    
    s3_resource = boto3.resource('s3')
    for key in spotify_key:
        copy_source = {
            'Bucket' : Bucket,
            'Key' : key
        }
        s3_resource.meta.client.copy(copy_source,Bucket,'raw-data/processed/' + key.split('/')[-1])
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

3. **Data Transformation** : A transformation function is implemented within AWS Lambda to process the raw data stored in S3.The function cleans, structures, and converts the raw data into a normalized format suitable for analytics.Transformation steps may include:Filtering and cleaning: Removing missing or inconsistent records.Standardization: Converting data types and formatting fields consistently.Enrichment: Adding calculated fields or additional metadata if needed.The transformed data is then stored in another S3 bucket for downstream processing.An S3 event trigger can be set up to automatically execute the transformation function whenever new raw data is uploaded.

4. **Data Loading into Snowflake** : 


   - A Snowflake table is created to store the transformed data from the S3 bucket. The schema is designed based on the data structure to ensure efficient querying and analysis.
    The data loading process is fully automated using Snowpipe, which enables real-time or near real-time ingestion.

   - Snowpipe’s auto-ingest feature is configured to detect new files in the S3 bucket as soon as they are uploaded.
    A notification mechanism (such as AWS S3 event notifications with Amazon SNS or SQS) is set up to trigger Snowpipe whenever new transformed data is added to S3.

   - Snowpipe uses a COPY INTO command to efficiently load the data from S3 into the Snowflake table, ensuring optimized performance.
    Data validation and monitoring mechanisms are implemented using Snowflake’s query history, metadata tables, and logging features to track ingestion status and troubleshoot issues if needed.

    - Once loaded, the data is ready for further analysis, reporting, and integration with BI tools or machine learning models.

### Conclusion : 
This automated pipeline ensures seamless data ingestion, minimal latency, and real-time availability of Spotify trending song data in Snowflake, making it highly efficient for business intelligence and analytics.
