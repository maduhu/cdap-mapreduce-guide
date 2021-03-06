===============================
Batch Data Processing with CDAP
===============================

`MapReduce <http://research.google.com/archive/mapreduce.html>`__ is the
most popular paradigm for processing large amounts of data in a reliable
and fault-tolerant manner. In this guide, you will learn how to batch
process data using MapReduce in the `Cask Data Application Platform
(CDAP) <http://cdap.io>`__.

What You Will Build
===================

This guide will take you through building a 
`CDAP application <http://docs.cdap.io/cdap/current/en/developers-manual/building-blocks/applications.html>`__
that uses ingested Apache access log events to compute the top 10 client IPs in a
specific time-range and query the results. You will:

- Build a
  `MapReduce program <http://docs.cdap.io/cdap/current/en/developers-manual/building-blocks/mapreduce-jobs.html>`__
  to process Apache access log events;
- Use a
  `Dataset <http://docs.cdap.io/cdap/current/en/developers-manual/building-blocks/datasets/index.html>`__
  to persist results of the MapReduce program; and
- Build a
  `Service <http://docs.cdap.io/cdap/current/en/developers-manual/building-blocks/services.html>`__
  to serve the results via HTTP.

What You Will Need
==================

- `JDK 7 or 8 <http://www.oracle.com/technetwork/java/javase/downloads/index.html>`__
- `Apache Maven 3.1+ <http://maven.apache.org/>`__
- `CDAP SDK <http://docs.cdap.io/cdap/current/en/developers-manual/getting-started/standalone/index.html>`__

Let’s Build It!
===============

The following sections will guide you through building an application from scratch. If you
are interested in deploying and running the application right away, you can clone its
source code from this GitHub repository. In that case, feel free to skip the next two
sections and jump right to the 
`Build and Run Application <#build-and-run-application>`__ section.

Application Design
------------------

The application will assume that the Apache access logs are ingested
into a Stream. The log events can be ingested into a Stream continuously
in real-time or in batches; whichever way, it doesn’t affect the ability
of the MapReduce program to consume them.

The MapReduce program extracts the required information from the raw logs
and computes the top 10 Client IPs by traffic in a specific time range.
The results of the computation are persisted in a Dataset.

Finally, the application contains a Service that exposes an HTTP
endpoint to access the data stored in the Dataset.

.. image:: docs/images/app-design.png
   :width: 8in
   :align: center

Implementation
--------------

The first step is to construct our application structure. We will use a
standard Maven project structure for all of the source code files::

  ./pom.xml
  ./src/main/java/co/cdap/guides/ClientCount.java
  ./src/main/java/co/cdap/guides/CountsCombiner.java
  ./src/main/java/co/cdap/guides/IPMapper.java
  ./src/main/java/co/cdap/guides/LogAnalyticsApp.java
  ./src/main/java/co/cdap/guides/TopClientsMapReduce.java
  ./src/main/java/co/cdap/guides/TopClientsService.java
  ./src/main/java/co/cdap/guides/TopNClientsReducer.java

The CDAP application is identified by the ``LogAnalyticsApp`` class. This
class extends an `AbstractApplication 
<http://docs.cdap.io/cdap/current/en/reference-manual/javadocs/co/cask/cdap/api/app/AbstractApplication.html>`__,
and overrides the ``configure()`` method to define all of the application components:

.. code:: java

  public class LogAnalyticsApp extends AbstractApplication {

    public static final String DATASET_NAME = "topClients";

    @Override
    public void configure() {
      setName("LogAnalyticsApp");
      setDescription("An application that computes the top 10 Client IPs from Apache access log data");
      addStream(new Stream("logEvents"));
      addMapReduce(new TopClientsMapReduce());
      try {
        DatasetProperties props = ObjectStores.objectStoreProperties(Types.listOf(ClientCount.class),
                                                                     DatasetProperties.EMPTY);
        createDataset(DATASET_NAME, ObjectStore.class, props);
      } catch (UnsupportedTypeException e) {
        throw Throwables.propagate(e);
      }
      addService(new TopClientsService());
    }
  }

The ``LogAnalytics`` application defines a new `Stream 
<http://docs.cdap.io/cdap/current/en/developers-manual/building-blocks/streams.html>`__
where Apache access logs are ingested.

The log events can be ingested into the CDAP stream. Once the data is
ingested, the events can be processed in real-time or batch. In our
application, we will process the events in batch using the
``TopClientsMapReduce`` program and compute the top 10 Client IPs in a
specific time-range.

The results of the MapReduce program is persisted into a Dataset; the
application uses the ``createDataset`` method to define the Dataset.
The ``ClientCount`` class defines the types used in the Dataset.

Finally, the application adds a service for querying the results from
the Dataset.

Let's take a closer look at the MapReduce program.

The ``TopClientsMapReduce`` extends an `AbstractMapReduce 
<http://docs.cdap.io/cdap/current/en/reference-manual/javadocs/co/cask/cdap/api/mapreduce/AbstractMapReduce.html>`__
class and overrides the ``configure()`` and ``initialize()`` methods:

-   ``configure()`` method configures a MapReduce, setting the program
    name, description and output Dataset.
-   ``initialize()`` method is invoked at runtime, before the MapReduce
    is executed. Here you can access the Hadoop job configuration through the
    ``MapReduceContext`` returned by ``getContext()``. Mapper, Reducer, and Combiner
    classes—as well as the intermediate data format—are set in this method.

.. code:: java

  public class TopClientsMapReduce extends AbstractMapReduce {

    @Override
    public void configure() {
      setName("TopClientsMapReduce");
      setDescription("MapReduce program that computes top 10 clients in the last 1 hour");
    }

    @Override
    public void initialize() throws Exception {
      MapReduceContext context = getContext();

      // Get the Hadoop job context, set Mapper, Reducer and Combiner.
      Job job = (Job) context.getHadoopJob();

      job.setMapOutputKeyClass(Text.class);
      job.setMapOutputValueClass(IntWritable.class);
      job.setMapperClass(IPMapper.class);

      job.setCombinerClass(CountsCombiner.class);

      // Number of reducer set to 1 to compute topN in a single reducer.
      job.setNumReduceTasks(1);
      job.setReducerClass(TopNClientsReducer.class);

      // Read events from last 60 minutes as input to the mapper.
      final long endTime = context.getLogicalStartTime();
      final long startTime = endTime - TimeUnit.MINUTES.toMillis(60);
      context.addInput(Input.ofStream("logEvents", startTime, endTime));
      context.addOutput(Output.ofDataset(LogAnalyticsApp.RESULTS_DATASET_NAME));
    }
  }

In this example, Mapper and Reducer classes are built by implementing
the `Hadoop APIs <http://hadoop.apache.org/docs/r2.3.0/api/org/apache/hadoop/mapreduce/package-summary.html>`__.

In the application, the Mapper class reads the Apache access log event
from the Stream and produces the Client IP and count as the intermediate
map output key and value:

.. code:: java

  public class IPMapper extends Mapper<LongWritable, Text, Text, IntWritable> {
    private static final IntWritable OUTPUT_VALUE = new IntWritable(1);

    @Override
    public void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
      // The body of the stream event is contained in the Text value
      String streamBody = value.toString();
      if (streamBody != null  && !streamBody.isEmpty()) {
        String ip = streamBody.substring(0, streamBody.indexOf(" "));
        // Map output Key: IP and Value: Count
        context.write(new Text(ip), OUTPUT_VALUE);
      }
    }
  }

The reducer class gets the Client IP and count from the MapReducer job and then
aggregates the count for each Client IP and stores it in a priority
queue. The number of reducers is set to 1, so that all results go into
the same reducer to compute the top 10 results. The top 10 results are
written to the MapReduce context in the cleanup method of the Reducer,
which is called once during the end of the task. Writing the results in
the context automatically writes the result to the output Dataset,
specified in the ``configure()`` method of the MapReduce program.

.. code:: java

  public class TopNClientsReducer extends Reducer<Text, IntWritable, byte[], List<ClientCount>> {

    private static final int COUNT = 10;
    private static final PriorityQueue<ClientCount> priorityQueue = new PriorityQueue<ClientCount>(COUNT);

    @Override
    protected void reduce(Text key, Iterable<IntWritable> values, Context context)
                          throws IOException, InterruptedException {
      // For each Key: IP, aggregate the Value: Count.
      int count = 0;
      for (IntWritable data : values) {
        count += data.get();
      }

      // Store the Key and Value in a priority queue.
      priorityQueue.add(new ClientCount(key.toString(), count));

      // Ensure the priority queue is always contains topN count.
      if (priorityQueue.size() > COUNT) {
        priorityQueue.poll();
      }
    }

    @Override
    protected void cleanup(Context context) throws IOException, InterruptedException {
      // Write topN results in reduce output. Since the "topN" (ObjectStore) Dataset is used as  
      // output the entries will be written to the Dataset without any additional effort.
      List<ClientCount> topNResults = Lists.newArrayList();
      while (priorityQueue.size() != 0) {
        topNResults.add(priorityQueue.poll());
      }
      context.write(TopClientsService.DATASET_RESULTS_KEY, topNResults);
    }
  }

Now that we have set the data ingestion and processing components, the
next step is to create a service to query the processed data.

The ``TopClientsService`` defines a simple HTTP RESTful endpoint to perform
this query and return a response:

.. code:: java

  public class TopClientsService extends AbstractService {

    public static final byte [] DATASET_RESULTS_KEY = {'r'};

    @Override
    protected void configure() {
      setName("TopClientsService");
      addHandler(new ResultsHandler());
    }

    public static class ResultsHandler extends AbstractHttpServiceHandler {

      @UseDataSet(LogAnalyticsApp.DATASET_NAME)
      private ObjectStore<List<ClientCount>> topN;

      @GET
      @Path("/results")
      public void getResults(HttpServiceRequest request, HttpServiceResponder responder) {

        List<ClientCount> result = topN.read(DATASET_RESULTS_KEY);
        if (result == null) {
          responder.sendError(404, "Result not found");
        } else {
          responder.sendJson(200, result);
        }
      }
    }
  }

Build and Run Application
=========================

The ``LogAnalyticsApp`` can be built and packaged using the Apache Maven command::

  $ mvn clean package

Note that the remaining commands assume that the ``cdap-cli.sh`` script is
available on your PATH. If this is not the case, please add it::

  $ export PATH=$PATH:<CDAP home>/bin

If you haven't already started a standalone CDAP installation, start it with the command::

  $ cdap.sh start

We can then deploy the application to the standalone CDAP installation::

  $ cdap-cli.sh load artifact target/cdap-mapreduce-guide-<version>.jar
  $ cdap-cli.sh create app LogAnalyticsApp cdap-mapreduce-guide <version> user

Next, we will send some sample Apache access log event into the stream
for processing::

  $ cdap-cli.sh send stream logEvents \'255.255.255.185 - - [23/Sep/2014:11:45:38 -0400] "GET /cdap.html HTTP/1.0" 200 190 "Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 5.1)"\'
  $ cdap-cli.sh send stream logEvents \'255.255.255.185 - - [23/Sep/2014:11:45:38 -0400] "GET /tigon.html HTTP/1.0" 200 102 "Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 5.1)"\'
  $ cdap-cli.sh send stream logEvents \'255.255.255.185 - - [23/Sep/2014:11:45:38 -0400] "GET /coopr.html HTTP/1.0" 200 121 "Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 5.1)"\'
  $ cdap-cli.sh send stream logEvents \'255.255.255.182 - - [23/Sep/2014:11:45:38 -0400] "GET /tigon.html HTTP/1.0" 200 111 "Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 5.1)"\'
  $ cdap-cli.sh send stream logEvents \'255.255.255.182 - - [23/Sep/2014:11:45:38 -0400] "GET /tigon.html HTTP/1.0" 200 145 "Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 5.1)"\'

We can now start the MapReduce program to process the events that were
ingested::

  $ cdap-cli.sh start mapreduce LogAnalyticsApp.TopClientsMapReduce

The MapReduce program will take a couple of moments to process.

We can then start the ``TopClientsService`` and then query the processed
results::

  $ cdap-cli.sh start service LogAnalyticsApp.TopClientsService

  $ curl -w'\n' http://localhost:10000/v3/namespaces/default/apps/LogAnalyticsApp/services/TopClientsService/methods/results

Example output::

  [{"clientIP":"255.255.255.185","count":3},{"clientIP":"255.255.255.182","count":2}]

You have now learned how to write a MapReduce program to process events from
a Stream, write the results to a Dataset and query the results using a Service.

Related Topics
==============

- `Wise: Web Analytics <http://docs.cask.co/tutorial/current/en/tutorial2.html>`__ tutorial, part of CDAP

Extend This Example
===================

Now that you have the basics of MapReduce programs down, you can extend
this example by:

- Writing a `workflow 
  <http://docs.cask.co/cdap/current/en/developers-manual/building-blocks/workflows.html>`__
  to schedule this MapReduce every hour and process the previous hour's data
- Store the results in a Timeseries data to analyze trends

Share and Discuss!
==================

Have a question? Discuss at the `CDAP User Mailing List <https://groups.google.com/forum/#!forum/cdap-user>`__.

License
=======

Copyright © 2014-2015 Cask Data, Inc.

Licensed under the Apache License, Version 2.0 (the "License"); you may
not use this file except in compliance with the License. You may obtain
a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
