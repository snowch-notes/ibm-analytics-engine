Open SSH session

    ssh clsadmin@your-cluster-name

Setup Flink

    # Download Flink
    wget -c -O flink-1.4.0-hadoop27-scala_2.11.tgz \
      "http://www.apache.org/dyn/mirrors/mirrors.cgi?action=download&filename=flink/flink-1.4.0/flink-1.4.0-bin-hadoop27-scala_2.11.tgz"

    # Extract
    tar xf flink-1.4.0-hadoop27-scala_2.11.tgz
    
    
Start Flink session

    # Run a Flink session - TODO how to determine arguments?
    ./flink-1.4.0/bin/yarn-session.sh -d -n 4

    # View the Flink session running on yarn
    yarn application -list

Deploy Flink job
 
    # Download Apache license
    wget -O LICENSE-2.0.txt http://www.apache.org/licenses/LICENSE-2.0.txt

    # Save to HDFS
    hadoop fs -copyFromLocal LICENSE-2.0.txt hdfs:///user/clsadmin/

    # Verify the file was saved
    hadoop fs -ls hdfs:///user/clsadmin/LICENSE-2.0.txt

    # Run the flink word count
    ./flink-1.4.0/bin/flink run ./flink-1.4.0/examples/batch/WordCount.jar \
       --input "hdfs:///user/clsadmin/LICENSE-2.0.txt" \
       --output "hdfs:///user/clsadmin/license-word-count.txt"

    # Verify the output
    hadoop fs -cat hdfs:///user/clsadmin/license-word-count.txt

----
Flink session arguments:

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
