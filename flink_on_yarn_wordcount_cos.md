### Introduction

Run the Flink Word Count example on IBM Analytics Engine as a yarn application.  

In this example, Flink reads a license from COS S3 performs a word count and saves the output to COS S3.

IAE sets up S3 to be accessed with a cos:// prefix whereas Flink requires a s3:// prefix, so in this example we have to configure both endpoints.

### Prerequisites

These instructions assume that you have a IBM COS S3 endpoint configured with [HMAC authentication](https://console.bluemix.net/docs/services/cloud-object-storage/iam/service-credentials.html#service-credentials), i.e.

Use the following steps to create a service credential:

 - Log in to the IBM Cloud console and navigate to your instance of Object Storage.
 - In the side navigation, click Service Credentials.
 - Click New credential and provide the necessary information. If you want to generate HMAC credentials, specify the following in the Add Inline Configuration Parameters (Optional) field: `{"HMAC":true}`
 - Click Add to generate service credential.
 - Open the new service credentials and find the attributes:
 
```
"cos_hmac_keys": {
  "access_key_id": "<Access Key ID>",
  "secret_access_key": "<Secret Access Key>"
},
```
 - The endpoint can be found by going to the side navigation, click Endpoint
 - select the **private** endpoint for your location.

### Configure IAE for COS

These instructions are taken from the [IAE docs](https://console.bluemix.net/docs/services/AnalyticsEngine/configure-COS-S3-object-storage.html#configuring-clusters-to-work-with-ibm-cos-s3-object-stores).

 - Open the Ambari console, and then the advanced configuration for HDFS.
 - Ambari dashboard > HDFS > Configs > Advanced > Custom core-site > Add Property
 - Add the properties and values.
 
 Note that the value for <servicename> can be any literal such as awsservice or myobjectstore.

```
fs.cos.<servicename>.access.key=<Access Key ID>
fs.cos.<servicename>.secret.key=<Secret Access Key>
fs.cos.<servicename>.endpoint=<EndPoint URL>
```

Also, in addition to the IAE documentation instructions, we need to set up to access S3 urls which are used by Flink

```
fs.s3a.access.key=<Access Key ID>
fs.s3a.secret.key=<Secret Access Key>
fs.s3a.endpoint=<EndPoint URL>
```

 - Save your changes and restart any affected services. The cluster will have access to your object store.

### Open SSH session

    ssh clsadmin@your-cluster-name
    
### Set some variables

    S3_ACCESS_KEY=your-access-key
    S3_SECRET_KEY=your-secret-key
    S3_ENDPOINT=your-endpoint
    S3_BUCKET=your-bucket
    
    # your-servicename as configured in IAE Ambari
    S3_SERVICENAME=your-servicename

### Setup Flink

    # Download Flink
    wget -c -O flink-1.4.0-hadoop27-scala_2.11.tgz \
      "http://www.apache.org/dyn/mirrors/mirrors.cgi?action=download&filename=flink/flink-1.4.0/flink-1.4.0-bin-hadoop27-scala_2.11.tgz"

    # Extract
    tar xf flink-1.4.0-hadoop27-scala_2.11.tgz
    
    FLINK_HOME=flink-1.4.0
    FLINK_LIB=$FLINK_HOME/lib/
    FLINK_CONF=$FLINK_HOME/conf/flink-conf.yaml
    
 ### Setup Flink S3
 
 For more information, see: https://ci.apache.org/projects/flink/flink-docs-release-1.4/ops/deployment/aws.html
    
    # Add S3 driver
    cp -f flink-1.4.0/opt/flink-s3-fs-hadoop-1.4.0.jar $FLINK_LIB
    
    # Add hadoop dependencies
    cp -f /usr/hdp/2.6.2.0-205/hadoop/hadoop-aws.jar $FLINK_LIB
    cp -f /usr/hdp/2.6.2.0-205/hadoop/lib/aws-java-sdk-s3-1.10.6.jar $FLINK_LIB
    cp -f /usr/hdp/2.6.2.0-205/hadoop/lib/aws-java-sdk-core-1.10.6.jar $FLINK_LIB
    cp -f /usr/hdp/2.6.2.0-205/hadoop/lib/aws-java-sdk-kms-1.10.6.jar $FLINK_LIB
    cp -f /usr/hdp/2.6.2.0-205/hadoop/lib/jackson-annotations-2.2.3.jar $FLINK_LIB
    cp -f /usr/hdp/2.6.2.0-205/hadoop/lib/jackson-core-2.2.3.jar $FLINK_LIB
    cp -f /usr/hdp/2.6.2.0-205/hadoop/lib/jackson-databind-2.2.3.jar $FLINK_LIB
    cp -f /usr/hdp/2.6.2.0-205/hadoop/lib/joda-time-2.9.4.jar $FLINK_LIB
    cp -f /usr/hdp/2.6.2.0-205/hadoop/lib/httpcore-4.4.4.jar $FLINK_LIB
    cp -f /usr/hdp/2.6.2.0-205/hadoop/lib/httpclient-4.5.2.jar $FLINK_LIB
    
    echo 'fs.hdfs.hadoopconf:  /etc/hadoop/conf' >> $FLINK_CONF
    
### Start Flink session

For more information, see: https://ci.apache.org/projects/flink/flink-docs-release-1.4/ops/deployment/yarn_setup.html

    # Run a Flink session
    # TODO how to determine what values to set for the arguments?
    #      see https://ci.apache.org/projects/flink/flink-docs-release-1.4/ops/deployment/yarn_setup.html#start-a-session)
    
    ./flink-1.4.0/bin/yarn-session.sh -d -n 4

    # View the Flink session running on yarn
    yarn application -list

### Deploy Flink job
 
    # Download Apache license
    wget -c -O LICENSE-2.0.txt http://www.apache.org/licenses/LICENSE-2.0.txt

    # Save to HDFS
    hadoop fs -copyFromLocal -f LICENSE-2.0.txt cos://${S3_BUCKET}.${S3_SERVICENAME}/LICENSE-2.0.txt

    # Verify the file was saved
    hadoop fs -ls cos://${S3_BUCKET}.${S3_SERVICENAME}/LICENSE-2.0.txt

    export HADOOP_CONF_DIR=/etc/hadoop/conf

    # Run the flink word count 
    # For more information, see: https://ci.apache.org/projects/flink/flink-docs-release-1.4/ops/deployment/yarn_setup.html
    
    ./flink-1.4.0/bin/flink run ./flink-1.4.0/examples/batch/WordCount.jar \
       --input "s3://${S3_BUCKET}/LICENSE-2.0.txt" \
       --output "s3://${S3_BUCKET}/license-word-count.txt"

    # Verify the output
    hadoop fs -cat cos://${S3_BUCKET}.${S3_SERVICENAME}/license-word-count.txt
