Setup Flink

    wget -c -O flink-1.4.0-hadoop27-scala_2.11.tgz \
      "http://www.apache.org/dyn/mirrors/mirrors.cgi?action=download&filename=flink/flink-1.4.0/flink-1.4.0-bin-hadoop27-scala_2.11.tgz"

    tar xf flink-1.4.0-hadoop27-scala_2.11.tgz
    
    
Start Flink session

    ./flink-1.4.0/bin/yarn-session.sh -d -n 4

    # view the Flink session running on yarn
    yarn application -list

Deploy Flink job
 
    wget -O LICENSE-2.0.txt http://www.apache.org/licenses/LICENSE-2.0.txt

    hadoop fs -copyFromLocal LICENSE-2.0.txt hdfs:///user/clsadmin/

    # verify the file was uploaded
    hadoop fs -ls hdfs:///user/clsadmin/LICENSE-2.0.txt

    # run the flink word count
    ./flink-1.4.0/bin/flink run ./flink-1.4.0/examples/batch/WordCount.jar \
       --input "hdfs:///user/clsadmin/LICENSE-2.0.txt" \
       --output "hdfs:///user/clsadmin/license-word-count.txt"

    # verify the output
    hadoop fs -cat hdfs:///user/clsadmin/license-word-count.txt
