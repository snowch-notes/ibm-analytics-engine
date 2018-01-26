### Introduction

Run the Flink Word Count example on IBM Analytics Engine as a yarn application

WARNING - these instructions are currently broken.  The error message is:

```
Caused by: org.apache.flink.runtime.client.JobExecutionException: Cannot initialize task 'DataSink 
   (CsvOutputFormat (path: s3://xxxxx/license-word-count.txt, delimiter:  ))': 
   doesBucketExist on csnow-us-geo: org.apache.flink.fs.s3hadoop.shaded.com.amazonaws.AmazonClientException: 
   No AWS Credentials provided by BasicAWSCredentialsProvider EnvironmentVariableCredentialsProvider 
   SharedInstanceProfileCredentialsProvider : org.apache.flink.fs.s3hadoop.shaded.com.amazonaws.SdkClientException: 
   Unable to load credentials from service endpoint
```

### Configure IAE for COS

 - Open the Ambari console, and then the advanced configuration for HDFS.
 - Ambari dashboard > HDFS > Configs > Advanced > Custom core-site > Add Property
 - Add the properties and values.
 
 Note that the value for <servicename> can be any literal such as awsservice or myobjectstore.

```
fs.cos.<servicename>.access.key=<Access Key ID>
fs.cos.<servicename>.secret.key=<Secret Access Key>
fs.cos.<servicename>.endpoint=<EndPoint URL>
```

Also, set up S3 urls which are needed by Flink

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
    
    # Set the S3 driver parameters
    #echo s3.access-key: $S3_ACCESS_KEY >> $FLINK_CONF
    #echo s3.secret-key: $S3_SECRET_KEY >> $FLINK_CONF
    #echo s3.endpoint: $S3_ENDPOINT >> $FLINK_CONF
    
### Start Flink session

    # Run a Flink session - TODO how to determine arguments (see below)?
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
    ./flink-1.4.0/bin/flink run ./flink-1.4.0/examples/batch/WordCount.jar \
       --input "s3://${S3_BUCKET}/LICENSE-2.0.txt" \
       --output "s3://${S3_BUCKET}/license-word-count.txt"

    # Verify the output
    hadoop fs -cat cos://${S3_BUCKET}.${S3_SERVICENAME}/license-word-count.txt

----
### Flink session arguments:

```
Usage:
   Required
     -n,--container <arg>   Number of YARN container to allocate (=Number of Task Managers)
   Optional
     -D <arg>                        Dynamic properties
     -d,--detached                   Start detached
     -jm,--jobManagerMemory <arg>    Memory for JobManager Container [in MB]
     -nm,--name                      Set a custom name for the application on YARN
     -q,--query                      Display available YARN resources (memory, cores)
     -qu,--queue <arg>               Specify YARN queue.
     -s,--slots <arg>                Number of slots per TaskManager
     -tm,--taskManagerMemory <arg>   Memory per TaskManager Container [in MB]
     -z,--zookeeperNamespace <arg>   Namespace to create the Zookeeper sub-paths for HA mode
 ```
