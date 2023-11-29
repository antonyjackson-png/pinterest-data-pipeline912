# Pinterest Data Pipeline

## Setting up Kafka on the EC2 instance

### Step 1: Creating an Amazon MSK Cluster
The primary VPC configuration is 3 zones.
There are 3 Brokers and the Broker Type is kafka.m5.large.
Each broker has 100GiB of EBS storage.
IAM role-based-authentication is enabled.
Within the cluster, TLS encryption is enabled.

### Step 2: Creating an EC2 client
The EC2 instance is a t2.micro AWS Linux machine.
The IAM access role is as follows:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": [
                    "ec2.amazonaws.com",
                    "kafkaconnect.amazonaws.com"
                ],
                "AWS": "arn:aws:iam::{aws_account_id}:role/{name_of_iam_access_role}"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

### Step 3: Configuring the EC2 client
Start by installing java:

```
sudo yum install java-1.8.0
```

Then install Apache Kafka:

```
wget https://archive.apache.org/dist/kafka/2.8.1/kafka_2.12-2.8.1.tgz
tar -xzf kafka_2.12-2.8.1.tgz
```

In the newly created kafka directory, navigate to the /libs folder and download the IAM MSK authentication package:

```
wget https://github.com/aws/aws-msk-iam-auth/releases/download/v1.1.5/aws-msk-iam-auth-1.1.5-all.jar
```

In the ~/.bashrc file, set the CLASSPATH environment variable:

```
export CLASSPATH=/home/ec2-user/kafka_2.12-2.8.1/libs/aws-msk-iam-auth-1.1.5-all.jar
```

Navigate to the /bin folder and create the following client.properties file:

```
# Sets up TLS for encryption and SASL for authN.
security.protocol = SASL_SSL

# Identifies the SASL mechanism to use.
sasl.mechanism = AWS_MSK_IAM

# Binds SASL client implementation.
sasl.jaas.config = software.amazon.msk.auth.iam.IAMLoginModule required awsRoleArn="{your aws access role arn}";

# Encapsulates constructing a SigV4 signature based on extracted credentials.
# The SASL client bound by "sasl.jaas.config" invokes this class.
sasl.client.callback.handler.class = software.amazon.msk.auth.iam.IAMClientCallbackHandler
```

### Step 4: Collect the Kafka BootstrapServerString
On the MSK cluster, go to the View Client Information tab and copy the BootstrapServerString

### Step 5: Create 3 topics:
Navigate to the /bin folder and create the following 3 topics: 
```
<your_UserId>.pin
```
```
<your_UserId>.geo
```
```
<your_UserId>.user
```

```
./kafka-topics.sh --bootstrap-server BootstrapServerString --command-config client.properties --create --topic <topic_name>

```
## Connecting the MSK Cluster to an S3 Bucket

### Step 1: Create a custom plugin with MSK Connect
In this step, download the following connector from Confluent and copy the file to the assigned S3 bucket

```
wget https://d1i4a15mxbxib1.cloudfront.net/api/plugins/confluentinc/kafka-connect-s3/versions/10.0.3/confluentinc-kafka-connect-s3-10.0.3.zip
```

### Step 2: Create the MSK Connector
In the Connector configuration settings, use the following configuration:
```
connector.class=io.confluent.connect.s3.S3SinkConnector
# same region as our bucket and cluster
s3.region=us-east-1
flush.size=1
schema.compatibility=NONE
tasks.max=3
# include nomeclature of topic name, given here as an example will read all data from topic names starting with msk.topic....
topics.regex=<TOPIC_ROOT>.*
format.class=io.confluent.connect.s3.format.json.JsonFormat
partitioner.class=io.confluent.connect.storage.partitioner.DefaultPartitioner
value.converter.schemas.enable=false
value.converter=org.apache.kafka.connect.json.JsonConverter
storage.class=io.confluent.connect.s3.storage.S3Storage
key.converter=org.apache.kafka.connect.storage.StringConverter
s3.bucket.name=<BUCKET_NAME>
```
## Send Data to the API

## Step 1: Integrate the API with Kafka
On the client EC2 instance run the following commands to install the REST proxy package:

```
sudo wget https://packages.confluent.io/archive/7.2/confluent-7.2.0.tar.gz
```
```
tar -xvzf confluent-7.2.0.tar.gz
```

Navigate to confluent-7.2.0/etc/kafka-rest and configure the kafka-rest.properties file with the followng information:
```
# Sets up TLS for encryption and SASL for authN.
client.security.protocol = SASL_SSL

# Identifies the SASL mechanism to use.
client.sasl.mechanism = AWS_MSK_IAM

# Binds SASL client implementation.
client.sasl.jaas.config = software.amazon.msk.auth.iam.IAMLoginModule required awsRoleArn="{your aws access role arn}";

# Encapsulates constructing a SigV4 signature based on extracted credentials.
# The SASL client bound by "sasl.jaas.config" invokes this class.
client.sasl.client.callback.handler.class = software.amazon.msk.auth.iam.IAMClientCallbackHandler
```

Note that the configuration is similar to the client.properties file from earlier, except client. is a prefix to all the commands.

### Step 2: Start the REST proxy
In order for messages to be consumed in MSK, start the REST proxy:

```
./kafka-rest-start /home/ec2-user/confluent-7.2.0/etc/kafka-rest/kafka-rest.properties
```

### Step 3: Send JSON Objects to the API
Streaming data is simulated with a Python script that emulates user posts on Pinterest.

The API is configured to receive JSON objects.  The ```user_posting_emulation.py``` script is modified to do so; in particular timestamp objects are converted to strings.

### Step 4: Check the Messages have been written to the S3 Bucket
If successful, a \topics directory will have been created in the S3 bucket and JSON objects for the <user-name>.pin, <user-name>.geo and <user-name>.user topics can be accessed and examined. 

## Batch Processing with Databricks
This step involves writing a Python notebook, which I have given the title ```Milestone 6 Task 2.ipynb``` to mount an S3 bucket to Databricks.

The notebook process the JSON messages into three dataframes:

```df_pin```
```df_geo```
```df_user```













