"All Roads Lead To Rome": Optimistic Recovery for Distributed Iterative Data Processing
=======================================================================================

In this repository, we provide code and instructions to repeat the experiments regarding the compensation of simulated failures in PageRank and ConnectedComponents using [Stratosphere](http://stratosphere.eu).


The Webbase2001 dataset can be downloaded in compressed form from the [Laboratory for Web Algorithmics](http://law.di.unimi.it/webdata/webbase-2001/), the Twitter social graph dataset can be obtained by contacting the [Max Planck Institute for Software Systems](http://twitter.mpi-sws.org/).

The modified variants of Stratosphere can be found in this github repository: [https://github.com/dimalabs/stratosphere-iterations/](https://github.com/dimalabs/stratosphere-iterations/). The version used for the performance tests can be found in the [sewen-iterations](https://github.com/dimalabs/stratosphere-iterations/tree/sewen-iterations) branch, the version used for the remaining experiments in the [ssc-iterations](https://github.com/dimalabs/stratosphere-iterations/tree/ssc-iterations) branch. Stratosphere jobs to preprocess and convert the datasets can be found in the [https://github.com/sscdotopen/dogfood](https://github.com/sscdotopen/dogfood) repository.





### PageRank on Webbase2001

Unpack the dataset and use the tools provided by the *Laboratory for Web Algorithmics* to bring into into edge-list format (text file with a pair of source and target vertex per line, separated by a tab). After that, use the Stratosphere jobs from the *doogfood* repository to prepare the input:

    bin/pact-client.sh run -j dogfood-1.0 EdgeListToAdjacencyList.jar 
      -w -a <numWorkers>hdfs:///webbase2001/webbase-2001-edges.tsv hdfs:///experiments/pagerank/webbase/adjacency/

    bin/pact-client.sh run -j dogfood-1.0-CreateMarkedVertexList.jar
      -w -a <numWorkers> hdfs:///webbase2001/webbase-2001-edges.tsv hdfs:///experiments/pagerank/webbase/vertices/

Now, the performance tests can be run using the *sewen*-branch of Stratosphere. We handcoded several optimizer decisions, therefore the program has to be executed as Nephele job:

    java -cp ".:./pact-runtime-0.2.jar:<stratosphere-dir>/lib/nephele-common-0.2.jar:<stratosphere-dir>/lib/pact-common-0.2.jar:
      <stratosphere-dir>/lib/commons-logging-1.1.1.jar:<stratosphere-dir>/lib/log4j-1.4.16.jar:<stratosphere-dir>/lib/guava-r09.jar:
      <stratosphere-dir>/lib/commons-codec-1.3.jar"
      eu.stratosphere.pact.runtime.iterative.compensatable.danglingpagerank.custom.CustomCompensatableDanglingPageRank 
      <numWorkers> <tasksPerWorker> hdfs:///experiments/pagerank/webbase/vertices/ hdfs:///experiments/pagerank/webbase/adjacency/ 
      hdfs:///results/pagerank/webbase/ranks/ <stratosphere-dir>/conf/ 256 256 512 60 115657290 25174808 
      0 1000 0 <checkpointLocation>

Grep for *IterationSynchronizationSinkTask* and *finishing iteration* in the log files to obtain the runtimes for individual iterations. 


As we computed the ranks now, we can use them to measure convergence for failures. We need to prepare add them to the input using a *dogfood* job:


    bin/pact-client.sh run -j dogfood-1.0-MatchWithRanks.jar 
      -w -a <numWorkers> hdfs:///experiments/pagerank/webbase/vertices/ hdfs:///results/pagerank/webbase/ranks/  
      hdfs:///experiments/pagerank/webbase/verticesWithRanks/ 


After that, we can run PageRank from the *ssc*-branch and simulate failures by supplying different values for the *<failingWorkers>*,  *<failingIteration>* parameters:

    java -cp ".:./pact-runtime-0.2.jar:<stratosphere-dir>/lib/nephele-common-0.2.jar:<stratosphere-dir>/lib/pact-common-0.2.jar:
      <stratosphere-dir>/lib/commons-logging-1.1.1.jar:<stratosphere-dir>/lib/log4j-1.4.16.jar:<stratosphere-dir>/lib/guava-r09.jar:
      <stratosphere-dir>/lib/commons-codec-1.3.jar"
      eu.stratosphere.pact.runtime.iterative.compensatable.danglingpagerank.CompensatableDanglingPageRank 
      <numWorkers> <tasksPerWorker> hdfs:///experiments/pagerank/webbase/vertices/ hdfs:///experiments/pagerank/webbase/adjacency/ 
      hdfs:///results/pagerank/webbase/verticesWithRanks/ <stratosphere-dir>/conf/ 256 256 512 60 115657290 25174808 
      <failingWorkers> <failingIteration> 0.5 <checkpointLocation>

Grep for *IterationHeadPactTask* and *head received global aggregate* in the log files to obtain statistics about the convergence


### ConnectedComponents on the Twitter graph

Again, we start with unpacking the dataset and preparing it via jobs from the *dogfood*-repository:

    bin/pact-client.sh run -j dogfood-1.0-EdgeListToAdjacencyList.jar 
      -w -a <numWorkers> hdfs:///experiments/twitter-icwsm2010/links-anon.txt 
      hdfs:///experiments/connectedcomponents/twitter-icwsm/adjacency/ true

    bin/pact-client.sh run -j dogfood-1.0-Uniquify.jar
      -w -a <numWorkers> hdfs:///experiments/twitter-icwsm2010/links-anon.txt 
      hdfs:///experiments/connectedcomponents/twitter-icwsm/vertices/

    bin/pact-client.sh run -j dogfood-1.0-Symmetrify.jar 
      -w -a <numWorkers> hdfs:///experiments/twitter-icwsm2010/links-anon.txt 
      hdfs:///experiments/connectedcomponents/twitter-icwsm/initialWorkset/


We use the connected components implementation from the *ssc*-branch to measure the performance with and without checkpointing:

    java -cp ".:<stratosphere-dir>/lib/nephele-common-0.2.jar:<stratosphere-dir>/lib/pact-common-0.2.jar:
      <stratosphere-dir>/lib/commons-logging-1.1.1.jar:<stratosphere-dir>/lib/log4j-1.4.16.jar:<stratosphere-dir>/lib/guava-r09.jar:
      <stratosphere-dir>/lib/commons-codec-1.3.jar" 
      eu.stratosphere.pact.runtime.iterative.compensatable.connectedcomponents.CompensatableConnectedComponents 
      <numWorkers> <tasksPerWorker> hdfs:///experiments/connectedcomponents/twitter-icwsm/vertices/
      hdfs:///experiments/connectedcomponents/twitter-icwsm/initialWorkset/
       hdfs:///experiments/connectedcomponents/twitter-icwsm/adjacency/ hdfs:///output-cc/ <stratosphere-dir>/conf/ 
      2048 1536 1024 0 1000 0

    java -cp ".:<stratosphere-dir>/lib/nephele-common-0.2.jar:<stratosphere-dir>/lib/pact-common-0.2.jar:
      <stratosphere-dir>/lib/commons-logging-1.1.1.jar:<stratosphere-dir>/lib/log4j-1.4.16.jar:<stratosphere-dir>/lib/guava-r09.jar:
      <stratosphere-dir>/lib/commons-codec-1.3.jar" 
      eu.stratosphere.pact.runtime.iterative.compensatable.connectedcomponents.CompensatableConnectedComponents 
      <numWorkers> <tasksPerWorker> hdfs:///experiments/connectedcomponents/twitter-icwsm/vertices/
      hdfs:///experiments/connectedcomponents/twitter-icwsm/initialWorkset/
       hdfs:///experiments/connectedcomponents/twitter-icwsm/adjacency/ hdfs:///output-cc/ <stratosphere-dir>/conf/ 
      2048 1536 1024 0 1000 0 <checkpointLocation>

Again, grepping for *IterationSynchronizationSinkTask* and *finishing iteration* in the log files gives the runtimes for individual iterations. 

    
After that we can again simulate failures by supplying different values for the *<failingWorkers>*,  *<failingIteration>* parameters:

    java -cp ".:<stratosphere-dir>/lib/nephele-common-0.2.jar:<stratosphere-dir>/lib/pact-common-0.2.jar:
      <stratosphere-dir>/lib/commons-logging-1.1.1.jar:<stratosphere-dir>/lib/log4j-1.4.16.jar:<stratosphere-dir>/lib/guava-r09.jar:
      <stratosphere-dir>/lib/commons-codec-1.3.jar" 
      eu.stratosphere.pact.runtime.iterative.compensatable.connectedcomponents.CompensatableConnectedComponents 
      <numWorkers> <tasksPerWorker> hdfs:///experiments/connectedcomponents/twitter-icwsm/vertices/
      hdfs:///experiments/connectedcomponents/twitter-icwsm/initialWorkset/
       hdfs:///experiments/connectedcomponents/twitter-icwsm/adjacency/ hdfs:///output-cc/ <stratosphere-dir>/conf/ 
      2048 1536 1024 <failingWorkers> <failingIteration> 0.5 

Grep for *IterationHeadPactTask* and *head received global aggregate* in the log files to obtain statistics about the convergence
