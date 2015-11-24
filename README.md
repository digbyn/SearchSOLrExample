# SearchSOLrExample
Simple Search Index creation with CDH SOLr

Search Example on QuickStart VM:

Please find below my quick notes on creating a simple search app in CDH using Solr.

For the purpose of this search example I am going to index an existing file within HDFS so this will be a 'batch' index of data and not a real-time index using flume. I have been following all the Cloudera Live tutorial exercises and performing all the exercises on my local instance of the Quickstart VM. You can do the same or follow the examples on any other instance/cluster. Some tutorial data files may need to be installed.

I am pretty much following the Solr Search example within the Cloudera Live Tutorials, section 4, found here:
http://www.cloudera.com/content/www/en-us/developers/get-started-with-hadoop-tutorial/exercise-4.html

Mark Brooks has also created a sample search application configuration guide found here which I have incorporated. Special thanks to Mark for his assistance and content.
https://wiki.cloudera.com/pages/viewpage.action?title=Simple+Cloudera+Search+Example&spaceKey=~mbrooks

During this exercise I had to perform Yarn tuning....just like our class exercise with Michael Ernest, the batch Map/Reduce jobs on my QuickstartVM would not run under the current config so this was a double exercise for me and a real world test of Yarn tuning....thank you Michael! If you shut down some unnecessary services and bump up the max container numbers for Yarn things should work ok.

Search Example:

DO NOT OPERATE AS ROOT, use another user, in my case, 'cloudera' as I am using the quickstart vm.
My hostname is also quickstart, important.

[cloudera@quickstart sample07]$ hostname
quickstart.cloudera

Setup a directory you want to use to store the search configs.
$ export PROJECT_HOME=~/sample07

Run the solrctl command to create solr configs
$ solrctl  instancedir --generate $PROJECT_HOME

You may have some existing search configs installed. Go here to see if you have any examples already running..
/opt/examples/flume

[cloudera@quickstart sample07]$ pwd
/home/cloudera/sample07
[cloudera@quickstart sample07]$

Alternate way of generating a sample config with this command.
solrctl --zk quickstart:2181/solr instancedir --generate sample07

Within the project_home conf folder, edit the file schema.xml and change the the following values. The schema.xml is a sample so we need to reference our data file, 'sample07'.

Replace the <fields>, <uniqueKey> and <copyField> elements in $PROJECT_HOME/conf/schema.xml with these values: 

<fields>
   <field name="code" type="string" indexed="true" stored="true"/>
   <field name="description" type="string" indexed="true" stored="true"/>
   <field name="salary" type="int" indexed="true" stored="true" /> 
   <field name="total_emp" type="int" indexed="true" stored="true" /> 
   <field name="text" type="text_general" indexed="true" stored="false" multiValued="true"/>
   <field name="_version_" type="long" indexed="true" stored="true"/> 
</fields>
 
<uniqueKey>code</uniqueKey>
 
<copyField source="code" dest="text"/>
<copyField source="description" dest="text"/>
<copyField source="salary" dest="text"/>
<copyField source="total_emp" dest="text"/>

Now we need to create a morphline etl file.
Create a file named "morphline1.conf" in the $PROJECT_HOME directory with this text, which will parse the data-file into records and fields, fix some non-numeric data and load the records into Solr.

Make sure to replace the  hostname in the zkHost field with the hostname of a Zk Server.   Don't use "localhost" as the Zk hostname will be used on data nodes during the MapReduce-based batch indexing process.


SOLR_LOCATOR : {
 
  collection : Sample-07-Collection
 
  # ZooKeeper ensemble -- set this to your cluster's Zk hostname(s)
  zkHost : "quickstart:2181/solr"
}
 
morphlines : [
  {
    id : morphline1
    importCommands : ["org.kitesdk.**", "org.apache.solr.**"]
 
    commands : [
 
     # Read the CSV data
     {
        readCSV {
          separator : "\t"
          columns : ["code","description","total_emp","salary"]
          ignoreFirstLine : false
          trim : false
          charset : UTF-8
        }
      }
 
      # Replace "*" salary data with a numeric value
      {
        if {
          conditions : [
            { equals { salary : ["*"] } }
          ]
             then : [
            { logDebug { format : "Converting non-numeric salary to 0"  } }
            { setValues { salary : "0" } }
            ]
        }
      }
 
      { sanitizeUnknownSolrFields { solrLocator : ${SOLR_LOCATOR} } }
 
      # load the record into a Solr server or MapReduce Reducer.
      { loadSolr { solrLocator : ${SOLR_LOCATOR} } }
 
    ]
  }
]

Create the Solr instance dir

Execute this command to create the Solr instance directory:

[cloudera@quickstart sample07]$ solrctl --zk quickstart:2181/solr instancedir --create Sample-07-Collection $PROJECT_HOME
Uploading configs from /home/cloudera/sample07/conf to quickstart:2181/solr. This may take up to a minute.
[cloudera@quickstart sample07]$

Create the Solr Collection

Execute this command to create the Solr Collection. Note the "-s" argument defines the number of shards, which should correspond to the number of Solr Server instances you have. In my case I have Solr Servers deployed on two nodes:

[cloudera@quickstart sample07]$ solrctl --zk quickstart:2181/solr collection --create Sample-07-Collection -s 1



Check the Collection in HUE

Go into HUE and click on Search -> Indexes -> Sample-07-Collection, you should see it there configured...but its empty. eg.

HUE http://quickstart.cloudera:8888/indexer/#manage

Solr Admin http://quickstart.cloudera:8983/solr/#/

http://quickstart.cloudera:8983/solr/#/~cloud



Copy the example log4j.properties file into the $PROJECT_HOME directory:

Get a sample log4j.properties file from this folder -> /usr/share/doc/search-1.0.0+cdh5.4.2+0/examples/solr-nrt

cp log4j.properties /home/cloudera/sample07

Setup HDFS

I am on the quickstart VM and am using the already provisioned user 'cloudera'.

Here I create a specific folder and set some open permissions to store my results from the batch indexing process.

[cloudera@quickstart solr-nrt]$ sudo -u hdfs hadoop fs -mkdir /user/cloudera/sample07

[cloudera@quickstart solr-nrt]$ sudo -u hdfs hadoop fs -chmod 777 /user/cloudera/sample07



Perform a dry-run batch load process with the following command if on the quickstart VM:

hadoop jar /usr/lib/solr/contrib/mr/search-mr-*-job.jar org.apache.solr.hadoop.MapReduceIndexerTool -D 'mapred.child.java.opts=-Xmx500m' --log4j $PROJECT_HOME/log4j.properties --morphline-file $PROJECT_HOME/morphline1.conf --output-dir hdfs://quickstart:8020/user/cloudera/sample07/  --verbose --go-live --zk-host quickstart:2181/solr --collection Sample-07-Collection --dry-run   hdfs://quickstart:8020/user/hive/warehouse/sample_07



Kick ass it works!

4989 [main] INFO  org.apache.solr.hadoop.MapReduceIndexerTool  - Done. Indexing 1 files in dryrun mode took 9.0903048E7 secs

4989 [main] INFO  org.apache.solr.hadoop.MapReduceIndexerTool  - Success. Done. Program took 1.67410598E9 secs. Goodbye.

Run it again without the --dry-run argument:

FAIL:

956  [main] INFO  org.apache.solr.cloud.ZkController  - Write file /tmp/1447964164773-0/clustering/carrot2/stc-attributes.xml

957  [main] INFO  org.apache.solr.cloud.ZkController  - Write file /tmp/1447964164773-0/clustering/carrot2/kmeans-attributes.xml

958  [main] INFO  org.apache.solr.cloud.ZkController  - Write file /tmp/1447964164773-0/schema.xml

1047 [main] INFO  org.apache.solr.hadoop.MapReduceIndexerTool  - Indexing 1 files using 1 real mappers into 1 reducers

Error: Java heap space

Error: GC overhead limit exceeded

Error: Java heap space

178730 [main] ERROR org.apache.solr.hadoop.MapReduceIndexerTool  - Job failed! jobName: org.apache.solr.hadoop.MapReduceIndexerTool/MorphlineMapper, jobId: job_1447794135351_0001

[cloudera@quickstart solr-nrt]$ free -m

             total       used       free     shared    buffers     cached

Mem:          9890       8968        922          0        164       2213

-/+ buffers/cache:       6589       3300

Swap:         8191          0       8191

[cloudera@quickstart solr-nrt]$



Perform Yarn Tuning:

Upon investigation with logs Yarn tuning is out of whack, unable to create containers to run job. Testing if Hadoop works though.

Make sure your YARN yarn.nodemanager.resource.memory-mb  is set to probably at least 4GB (two allow at least 2 2GB containers).
Set yarn.nodemanager.resource.cpu-vcores  to as many cores as you can so that at least each container has 1 core, in my case I set to 4.
YARN container max memory is the key.
Re-run 
IT WORKS!

hadoop jar /usr/lib/solr/contrib/mr/search-mr-*-job.jar org.apache.solr.hadoop.MapReduceIndexerTool -D 'mapred.child.java.opts=-Xmx500m' --log4j $PROJECT_HOME/log4j.properties --morphline-file $PROJECT_HOME/morphline1.conf --output-dir hdfs://quickstart:8020/user/cloudera/sample07/  --verbose --go-live --zk-host quickstart:2181/solr --collection Sample-07-Collection hdfs://quickstart:8020/user/hive/warehouse/sample_07

1047 [main] INFO  org.apache.solr.hadoop.MapReduceIndexerTool  - Indexing 1 files using 1 real mappers into 1 reducers

30511 [main] INFO  org.apache.solr.hadoop.MapReduceIndexerTool  - Done. Indexing 1 files using 1 real mappers into 1 reducers took 9.8214881E9 secs

30531 [main] INFO  org.apache.solr.hadoop.GoLive  - Live merging of output shards into Solr cluster...

30536 [pool-4-thread-1] INFO  org.apache.solr.hadoop.GoLive  - Live merge hdfs://quickstart.cloudera:8020/user/cloudera/sample07/results/part-00000 into http://quickstart.cloudera:8983/solr

31207 [main] INFO  org.apache.solr.hadoop.GoLive  - Committing live merge...

31226 [main] INFO  org.apache.solr.common.cloud.SolrZkClient  - Using default ZkCredentialsProvider

31227 [main] INFO  org.apache.solr.common.cloud.ConnectionManager  - Waiting for client to connect to ZooKeeper

31230 [main-EventThread] INFO  org.apache.solr.common.cloud.ConnectionManager  - Watcher org.apache.solr.common.cloud.ConnectionManager@6715294d name:ZooKeeperConnection Watcher:quickstart.cloudera:2181/solr got event WatchedEvent state:SyncConnected type:None path:null path:null type:None

31230 [main] INFO  org.apache.solr.common.cloud.ConnectionManager  - Client is connected to ZooKeeper

31230 [main] INFO  org.apache.solr.common.cloud.SolrZkClient  - Using default ZkACLProvider

31233 [main] INFO  org.apache.solr.common.cloud.ZkStateReader  - Updating cluster state from ZooKeeper...

31495 [main] INFO  org.apache.solr.hadoop.GoLive  - Done committing live merge

31496 [main] INFO  org.apache.solr.hadoop.GoLive  - Live merging of index shards into Solr cluster took 3.21394208E8 secs

31496 [main] INFO  org.apache.solr.hadoop.GoLive  - Live merging completed successfully

31496 [main] INFO  org.apache.solr.hadoop.MapReduceIndexerTool  - Succeeded with job: jobName: org.apache.solr.hadoop.MapReduceIndexerTool/MorphlineMapper, jobId: job_1448309590988_0001

31496 [main] INFO  org.apache.solr.hadoop.MapReduceIndexerTool  - Success. Done. Program took 1.05121167E10 secs. Goodbye.


Go back into HUE, Search application and review the search index for the Sample07 collection. You should now see indexed data. You can now create a drag and drop search dashboard over the basic data. Nothing too exciting here but hey it works, time to index something cool.

http://quickstart.cloudera:8888/search/browse/Sample-07-Collection

To view whats coming/possible with Spark Notebooks and better search go here:

http://gethue.com/bay-area-bikeshare-data-analysis-with-search-and-spark-notebook/

Thanks, Digby Norris.
