---
layout: post
title:  Extending RDD for fun and profit
date:   2014-12-01 09:34:33
categories: bigdata spark rdd

---

You will need to extend RDD for mostly two reasons, you want to handle custom input format i.e. different underlying storage,
or you want to introduce an operation on RDD. For the second case,
you can build the operation on top of existing combinators like map, flatMap, filter etc.
For example, if I wanted the typical word counting logic available as a function on RDD,
or count all the elements which fulfil a specific criteria, we can do it as,

{% highlight scala %}
object RDDImplicits {
 implicit class RichRDD[T: ClassTag](rdd: RDD[T]) {

   def countEachElement = {
     rdd
     .map(element => (element, 1))
     .reduceByKey((value1, value2) => value1 + value2)
   }

   def countWhere(f: T => Boolean): Long = {
    rdd.filter(f).count()
   }
}
{% endhighlight %}


If you are new to Scala, the above function is an implicit method(equivalent to extension methods) and is available on RDD.
Now you can call those methods on any RDD as if it were a method on RDD itself.

Before going into details of RDD interface, some terminology brush up

* [**RDD(Resilient Distributed Datasets)**](http://spark.apache.org/docs/latest/programming-guide.html#resilient-distributed-datasets-rdds) - holds set of partition

* **Partition/split** - data splits which are distributed in the underlying storage, equivalent of splits in HDFS

* **Dependency** - the parent RDD from which this RDD is created, can be one-to-one aka Narrow dependency and Shuffle dependency.
In Narrow dependency, each partition from parent RDD is mapped to partition of new RDD.
In case of shuffle dependency, a shuffle of data is required between partitions.

If you want to extend RDD for the purpose of supporting some different storage,
or the existing combinators are not good enough for you and you want to introduce a new combinator, then
you will need to introduce a new custom RDD to suit your needs. Before that, lets understand the RDD interface.
The interface has following abstract methods which every subclass of RDD has to implement.

{% highlight scala %}

def compute(split: Partition, context: TaskContext): Iterator[T]

def getPartitions: Array[Partition]
{% endhighlight %}

There are other methods also like ```getDependencies, getPreferredLocations``` but we won't go into those in this post.

Every RDD has set of partitions with it. The computation is applied on each partition in parallel.
You can implement custom partition for your custom RDD to suit your needs.
Partition represents the underlying splice/split of data.
The compute method of RDD returns an iterator from the partition it has been provided.
This is the place where we will apply the transformation on the iterator of the partition.
Since RDD operations are chained, they create a DAG of transformations, and a new RDD has dependency on its parent.
This dependency can be of two types, NarrowDependecy, ie One-to-One dependency or wide dependency(occurs due to a shuffle).

Lets look at one of the Simplest RDD, MappedRDD.
Since the ```map``` operation does not need to shuffle the data, the MappedRDD has a one-to-one dependency on its parent.
Hence each of the partition of the parent is mapped the transformation directly, without requiring any shuffle of data.
Hence the getPartitions method just returns the partitions of the parent.

{% highlight scala %}
override def getPartitions: Array[Partition] = firstParent[T].partitions
{% endhighlight %}


The compute method applies the given transformation on the partition and returns an iterator of the mapped partition.

{% highlight scala %}
override def compute(split: Partition, context: TaskContext) =
 firstParent[T].iterator(split, context).map(f)
{% endhighlight %}


Same is the case for FlatMappedRDD, FilteredRDD.
For FlatMappedRDD, we will flatMap on the iterator the given transformation,
and for the FilteredRDD we will apply filter on the iterator for computing the partition.

One more need for extending RDD is to support some different underlying storage format.
A very good example is existing HadoopRDD.
The HadoopRDD make use of the InputFormat class to read the records using RecordReader.
The [compute method](https://github.com/apache/spark/blob/master/core/src/main/scala/org/apache/spark/rdd/HadoopRDD.scala#L209)
just gives an iterator, where next element is asked from the underlying recordReader.

One more similar example is parallelizing a collection.
The ```parallelize``` method on the SparkContext object creates a RDD with the collection split into slices/partitions and they collectively represent the RDD allowing us applying parallel operations on it.

**Parallelizing a collection**

Lets say we want to parallelize a collection. We have to split it into set of slices/partitions/splits.
The number of partitions can be asked from the user, or can be set to a fixed value of defaultParallelism.
The ```getPartitions``` method of that RDD will just return partitions with split data in it.
The compute method will return an iterator from collection held by the partition.
The [actual implementation](https://github.com/apache/spark/blob/master/core/src/main/scala/org/apache/spark/rdd/ParallelCollectionRDD.scala#L96)
is quite elaborated since it also takes into account all the edge case that might occur for partitioning the data,
but the idea remains the same.

Hope this helps! Any feedback/comments are welcome.

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

[Github]:  https://github.com/rahulkavale
[Twitter]: https://twitter.com/RBKavale