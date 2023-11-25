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

<your_UserId>.pin
<your_UserId>.geo
<your_UserId>.user

```
./kafka-topics.sh --bootstrap-server BootstrapServerString --command-config client.properties --create --topic <topic_name>
```
















