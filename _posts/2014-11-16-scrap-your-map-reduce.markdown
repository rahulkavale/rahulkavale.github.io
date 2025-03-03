---
layout: post
title:  Scrap your MapReduce! (Or, Introduction to Apache Spark)
date:   2014-11-16 09:34:33
categories: bigdata spark map-reduce

---

You might have heard that Spark recently broke the previous record for sort benchmark. That drove lot of attention towards Spark. In this post, I will be explaining briefly what is Spark and advantages of using it, also mentioning how it is more better in terms of performance  compared to MapReduce. Lets explore Spark.


Apache Spark is a ***cluster computing engine***. It abstracts away the underlying distributed storage and cluster management aspects, making it possible to plug in a lot of specialized storage and cluster management tools. Spark support HDFS, Cassandra, local storage, S3, even tradtional database for the storage layer. Spark can work with cluster management tools like YARN, Mesos. It also has its own standalone mode for cluster management purpose. Lets look at Apache Spark in detail, and I will try to address some of questions that  a common for Hadoop user/enthusiast will be curios about.

* **What does it replace in existing ecosystem?**

	 Actually Spark does not replace anything in traditional Hadoop ecosystem. Since Hadoop 2, its just yet another application that runs inside a <a href="http://stackoverflow.com/questions/14365218/what-is-a-container-in-yarn">YARN container</a>. Hence it fits very well inside the Hadoop eco-system. It offers us a consise, testable, readble, maintanable way to program, freeing us from writing painful MapReduce jobs also offering significant amount of [performance gain](http://databricks.com/blog/2014/10/10/spark-petabyte-sort.html). We will look in details each of these points.
	 
* **Some properties of "Big Data"**
	
	Big data is inherently immutable, meaning it is not supposed to updated once generated.
	
	The data itself is mostly structured or semi structured.
	
    Since the enormous size of data, commodity hardware makes more sense for its storage/computation, and hence its always distributed and powered by not so high end hardware, making the data distributed across cluster of many such machines, and as we know the distributed nature makes the programming complicated.
	

*  **Immutability and MapReduce model**
	
	The Map reduce model lacks to exploit the immutable nature of the data. A non trivial MapReduce(MR) job contains set of MR phases and in the name of fault tolerance, the intermediate results are persisted to disk causing lot of IO, causing a serious performance hit.
	
	
*  **Pain points with MapReduce model**

	MR code requires a significant amount of boilerplate.

	The programmer is required to think only in terms of basic operations of Map and reduce and mapping every problem to MR is not at trivial. Common operations like joining two data sets require significant amount of work.
	
	MR model is not suitable for iterative processing.
	
	The programmer is not at all transparent to the distributed nature of data, and often needs to think for optimizations like Map side reduce, Map side join etc.

	Although there are tools for addressing some of above mentioned pain points in terms of programming MR job by providing some higher level of abstraction, like Cascading, Scalding, Hive(SQL interface to MR) they do not improve any performance aspect as they are bound by the underlying MR jobs.
		
	Hadoop was meant for ***batch*** kind of operations.
	
Lets look at programming model for Spark, and see how it aims to solve above mentioned problems. 
	
*  **How does Spark's programming model look like?**

	 The main abstraction for computations in Spark is [Resilient Distributed Datasets(RDD)](https://www.cs.berkeley.edu/~matei/papers/2012/nsdi_spark.pdf). Due to its simplified programming interface, it unifies computational styles which were spread out in otherwise traditional Hadoop stack.

	eg. provides SQL like interface through ***SparkSQL***, streaming from ***Spark Streaming***. Iterative processing like machine learning , graph processing is possible via ***MLib***, ***graphX***. Spark also provides programming interface in languages including Scala, Java, Python.
	The abstraction of RDD, due to its properties also is a reason for much more performant nature of Spark. Lets see how.

* **What is RDD?**
	
	RDD stands for Resilient Distributed Dataset. It forms the basic abstraction on which Spark programming model works.
	
	RDD is ***immutable***.

    > This is a very important point, because even HDFS is write once, read many times/append only store, making it immutable but the MapReduce model makes it impossible to exploit this fact for improving performance.

    RDD is ***lazily evaluated***
	
    > RDD is not materialized unless an action(terminal operation) is called on it. This means when a terminal operation is called, Spark is aware of DAG of transformation it has to do on the data, making it repeatable operation in case of failures, and hence the fault tolerance aspect becomes trivial. It also allows to have some optimizations possible on computations steps which were otherwise impossible to guess.

	It can be thought of as ***Distributed collections***. The programming interface almost makes distributed nature of underlying data transparent.

	It can be created via, parallelizing a normal collection of values,transforming an existing RDD by applying a transformation function, reading from a persistent data store like HDFS.


* **What it means to operate on RDD vs traditional MapReduce?**
	
	RDD abstracts us away from traditional map-reduce style programs, giving us interface of a collection(which is distributed), and hence lot of operations which required quite a boilerplate in MR are now just collection operations, e.g. groupBy, joins, count, distinct, max, min etc.
It also allows us to iterative processing quite easily, by sharing RDD between operations.

	RDD can also be ***optionally cached*** giving quite a performance boost.

* **How is computation model when compared to MapReduce?**

	MR model is composed of following stages,
    
  > Map -> optional combine(Map side reduce) -> shuffle and sort -> Reduce
  
  It also allows us to have a custom partitioner to exploit partitioned nature of data.
	
  Welcome to Spark model, the beautiful thing about Spark is, it does not restrict us to traditional just Map and reduce operations. It allows us to apply collection like operations on a RDD, giving us another RDD. Since its just a RDD, it can queried via SQL interface, applied machine learning algorithms, and lot of fancy stuff.
Lets look at a word count example, done in Spark,

{% highlight scala %}
val input = sparkContext.textFile("path/to/input/file")
val words = input.flatMap(line => line.split(" "))
val wordsMappedToOne = words.map(word => (word, 1))
val wordOccrCount = wordsMappedToOne.reduceByKey((a, b) => a + b)
wordOccrCount.saveAsTextFile("path/to/output/file")
{% endhighlight %}
	
More examples can be found [here](https://github.com/rahulkavale/spark-examples). 
	
I have used the Scala interface for Spark. The code looks quite self-explanatory. Notice `sparkContext` is the way you specify the Spark configuration, and connect to the cluster. The remaining code are just containing collection operations. One important point to note here is, since RDD is lazily evaluated, no code is getting executed on the cluster till the point we actually ask Spark, in this case to save the output to the destination path. 
	There are two kind of operations allowed on RDD, First one is, set of transformations, these operations do not evaluate, but rather produce new RDD, with the transformation to be applied. Spark creates a DAG of these transformations.
e.g. map, flatMap, reduceByKey, groupByKey, join, etc.
Second one is, an action, these are terminal operations, and triggers actual calculation on the DAG which contains all the operations that are to be applied. 
e.g. count, collect, max etc.

* **What it means for a normal programmer?** 
		
	*	The code is just ***readable*** enough, to reason about it.

	*	The code is ***testable***, as it is just normal scala code, and a Spark cluster can be spawn in local mode to run the tests. This is very important as tradidionally it was very difficult to test MR code. 
	
	*   Spark also supports Python, Java apart form Scala.

	*	The code can be made up of normal domain models, wherever required.

	*	The Spark computation model also offers significant amount of ***performance***, reducing the latencies introduced as compared to traditional MR model.

	*	Spark unifies different ways to process data. Using SparkSQL, one can query the data by SQL queries whenever required, also allowing to apply normal collection-like operations.
		
	*	Map side reduce, or a combiner is not required at all, since the reduce operations are by default having a local aggregation at each map side.
	
	*	The code is testable! Yes this is this so important that I am mentioning it twice.

	*   Unifies various computation needs into a single place making the life easier and not needing to share data between different persistent stores just to do some sort of specialized processing.

	* 	Spark also gives a shell, which is very-very good for the exploration of data initailly before writing the job, which I guess is very very important.

	* 	We can use lot of already available libraries in our code, like Algebird, to do some fancy stuff.

	*	We will talk about debugging of Spark jobs in next post.


* **Why not use something like Cascading, Scalding?**

	 This comparison is actually not fair as Cascading, Scalding are libraries, where Spark is framework. These libraries actually introduce newer, richer abstractions over MR model. They don’t give any performance benefits.

	 In case of Spark, being a full fledged computation engine, it does not use MR framework but has its own computation model based on RDDs. It gives performance benefits apart from the point we mentioned above.
	 
  It also allows ***optional in memory processing*** of RDD giving incredible speedup. Apart from that, as we discussed, its storage agnostic, meaning it can be used to compute from data sources like local files, S3 storage, HDFS, JDBC data sources, Cassandra, and others.	

*	**How is normal development workflow like?**
	
	* Spark-shell usually comes handy here.

	* load the data from persistent store.

    * Explore the data, using normal functions, also explore the sampled data to understand 
structure of the data, understand the variance in data, etc

    * Model the workflow in terms of set of operations in the RDDs made out of sample data.

	* Understand possible partitioning schemes, like maybe data is date partitioned, hence operations can make use of that to reduce the network IO required for data shuffling, or a suitable partitioning which fits the problem.

	* Sharable datasets among various operations, should be cached, if required.

	* Test the output for correctness, test the business logic

	* Run the program on the proper data, to make sure it is performant enough.

	* Using local aggregations(what a combiner does in MR) whenever possible(i.e. avoid using groupBy, instead use reduceByKey or others depending on the requirement).

*   **Isn’t Spark about in-memory computations?**

	Often, I have seen people confusing Spark, as just in memory computation engine. Well, Spark is not at all just in-memory. It provides optional in-memory storage, mainly for performance boosts. To quote the official documentation, Almost all Spark operators perform external operations when data does not fit in memory. More generally, Spark’s operators are a strict superset of MapReduce. The user can very well specify the STORAGE_LEVELs which are memory-only, memory-and-disk, disk-only etc, which handle data spilling out of memory into disk. 

* **How is Spark more performant than Hadoop MR and family?**

	The intermediate result in MR were always persisted in order to enable fault tolerance aspect, costing a very high IO operation. While in case of Spark these result are not at all persisted unless the user explicitly does it. Enabling pipelining of operation resulting in great speedup.

	Usually network IO becomes a heavy bottleneck for distributed systems. Spark continues to have the code-parallel model, taking code to the data, in terms of serialized closures, reducing the network, this was also the case in Hadoop. Also, the local aggregation on results, help reduce the network IO in case of reduce operations.

	Spark also allows to persist RDD in memory and share it between different operations, giving a huge performance boost. In case of data not fitting in memory, the spill is taken in disk, and this is totally transparent to the programmer. In this case the performance is comparable to tradional processing.

To summarize,
<br>

<table border="line">
<tr> <td>Criteria</td><td>Map Reduce</td><td>Spark</td></tr>

<tr><td>Conciseness</td><td>Plain MR has a lot of boiler plate</td><td>Almost no boilerplate</td></tr>
<tr><td>Performance</td><td>High latency</td><td>very fast compared to MR</td></tr>
<tr><td>Testability</td><td>Possible via libraries, but non trivial</td><td>Very much easy</td></tr>
<tr><td>Iterative processing</td><td>Non trivial</td><td>straight forward</td></tr>
<tr><td>Exploration of data</td><td>Not possible easily</td><td>Spark shell allows quick and easy data exploration</td></tr>
<tr><td>SQL like interface</td><td>Via Hive</td><td>Build in as SparkSQL</td></tr>
<tr><td>Fault Tolerance</td><td>Inheranlty able to handle fault tolerance via persisting the results of each of phases</td><td>Exploits immutability of RDD to enable fault tolerance</td></tr>
<tr><td>Eco system</td><td>lots of tools available but integration is not quite seamless, requiring lot of effort for their seamless integration </td><td>Unifies lot of interfaces like SQL, stream processing etc into single abstraction of RDD</td></tr>
<tr><td>In memory computations</td><td>not possible</td><td>possible</td></tr>
</table>

<br>
These are some questions I wanted to cover for Apache Spark. Since I did not want the post to become overwhelming, I will be covering aspects like how fault tolerance is handled in Spark, what happens to job scheduling, a lifecycle of a job in Spark model, debugging a Spark job, how does shuffle work in Spark etc in next article. Thanks for your patience. Any questions/feedback is welcome!

<div class="social-share">
    <div class="tweet-button">
        <a class="tweet-button" href="https://twitter.com/share" class="twitter-share-button" data-via="yphalcombinator">Tweet</a>

        <!-- Put this just before the closing body tag -->
        <script>!function(d,s,id){var js,fjs=d.getElementsByTagName(s)[0];if(!d.getElementById(id)){js=d.createElement(s);js.id=id;js.src="//platform.twitter.com/widgets.js";fjs.parentNode.insertBefore(js,fjs);}}(document,"script","twitter-wjs");</script>
    </div>

    <div class="gplus-button">
        <!-- Google + -->
        <p class="gplus-button"><g:plusone size="medium"></g:plusone></p>

        <!-- Add this just before the closing body tag of your web page -->
        <script type="text/javascript">
          (function() {
            var po = document.createElement('script'); po.type = 'text/javascript'; po.async = true;
            po.src = 'https://apis.google.com/js/plusone.js';
            var s = document.getElementsByTagName('script')[0]; s.parentNode.insertBefore(po, s);
          })();
        </script>
    </div>
</div>

[Github]:   https://github.com/rahulkavale
[Twitter]: https://twitter.com/RBKavale