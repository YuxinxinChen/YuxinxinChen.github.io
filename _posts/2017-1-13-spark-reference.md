---
layout: post
title: "Spark Reference"
data: 2017-1-13
tags: [reading notes, spark]
comments: true
share: false
---

```scala
val lines = sc.textFile("/path/to/File")
val sparklines = lines.filter(line => line.contains("spark")
val shelllines = lines.filter(line => line.contains("shell") 
val sparkandshell= sparklines.union(shelllines)
println("Input had " + sparkandshell.count() + " concerning lines")
println("Here are 10 examples:")
sparkandshell.take(10).foreach(println)
```
RDDs also have a collect() function to retrieve the entire RDD. This can be useful if your program filters RDDs down to a very small size and you’d like to deal with it locally. Keep in mind that your entire dataset must fit in memory on a single machine to use collect() on it, so collect() shouldn’t be used on large datasets.  
Loading data into an RDD is lazily evaluated in the same way transformations are. So, when we call sc.textFile() , the data is not loaded until it is necessary. As with transformations, the operation (in this case, reading the data) can occur multiple times.

```scala
class SearchFunctions(val query: String) {
	def isMatch(s: String): Boolean = {
		s.contains(query)
	}
	def getMatchesFunctionReference(rdd: RDD[String]): RDD[String] = {
	// Problem: "isMatch" means "this.isMatch", so we pass all of "this"
		rdd.map(isMatch)
	}
	def getMatchesFieldReference(rdd: RDD[String]): RDD[String] = {
	// Problem: "query" means "this.query", so we pass all of "this"
		rdd.map(x => x.split(query))
	}
	def getMatchesNoReference(rdd: RDD[String]): RDD[String] = {
	// Safe: extract just the field we need into a local variable
		val query_ = this.query
	rdd.map(x => x.split(query_))
	}
}
```
If NotSerializableException occurs in Scala, a reference to a method or field in a nonserializable class is usually the problem. Note that passing in local serializable variables or functions that are members of a top-level object is always safe.

The map() transformation takes in a function and applies it to each element in the RDD with the result of the function being the new value of each element in the resulting RDD. The filter() transformation takes in a function and returns an RDD that only has elements that pass the filter() function.
It is useful to note that map() ’s return type does not have to be the same as its input type, so if we had an RDD String and our map() function were to parse the strings and return a Double , our input RDD type would be RDD[String] and the resulting RDD type would be RDD[Double] .

```scala
val input = sc.parallelize(List(1, 2, 3, 4))
val result = input.map(x => x * x)
println(result.collect().mkString(","))
```
The operation to do this is called flatMap() . As with map() , the function we provide to flatMap() is called individually for each element in our input RDD. Instead of returning a single element, we return an iterator with our return values. Rather than producing an RDD of iterators, we get back an RDD that consists of the elements from all of the iterators. A simple usage of flatMap() is splitting up an input string into words,

```scala
val lines = sc.parallelize(List("hello world", "hi"))
val words = lines.flatMap(line => line.split(" "))
words.first() // returns "hello"
```

RDD1.distinct(RDD2)
RDD1.union(RDD2)
RDD1.intersection(RDD2)
RDD1.subtract(RDD2)
require RDD1 and RDD2 are of the same type
cartesian(other) returns all possible pairs of (a,b) where a is in the source RDD and b is in the other RDD. Very expensive for large RDDs.

Action: reduce()
Similar to reduce() is fold() , which also takes a function with the same signature as needed for reduce() , but in addition takes a “zero value” to be used for the initial call on each partition. The zero value you provide should be the identity element for your operation; that is, applying it multiple times with your function should not change the value (e.g., 0 for +, 1 for *, or an empty list for concatenation).

Both fold() and reduce() require that the return type of our result be the same type as that of the elements in the RDD we are operating over. This works well for opera‐ tions like sum , but sometimes we want to return a different type. For example, when computing a running average, we need to keep track of both the count so far and the number of elements, which requires us to return a pair. We could work around this by first using map() where we transform every element into the element and the number 1, which is the type we want to return, so that the reduce() function can work on pairs.

The aggregate() function frees us from the constraint of having the return be the same type as the RDD we are working on. With aggregate() , like fold() , we supply an initial zero value of the type we want to return. We then supply a function to com‐ bine the elements from our RDD with the accumulator. Finally, we need to supply a second function to merge two accumulators, given that each node accumulates its own results locally.

```scala
val result = input.aggregate((0, 0))(
(acc, value) => (acc._1 + value, acc._2 + 1),
(acc1, acc2) => (acc1._1 + acc2._1, acc1._2 + acc2._2))
val avg = result._1 / result._2.toDouble 
```

resutl = (10,4), avg = 2.5

```scala
val result = input.combineByKey(
(v) => (v, 1),
(acc: (Int, Int), v) => (acc._1 + v, acc._2 + 1),
(acc1: (Int, Int), acc2: (Int, Int)) => (acc1._1 + acc2._1, acc1._2 + acc2._2)
).map{ case (key, value) => (key, value._1 / value._2.toFloat) }
result.collectAsMap().map(println(_))
```
(v) => (v, 1) //createCombiner
(acc: (Int, Int), v) => (acc._1 + v, acc._2 + 1) // mergeValue
(acc1: (Int, Int), acc2: (Int, Int)) => (acc1._1 + acc2._1, acc1._2 + acc2._2) // mergeCOmbiners
)

```scala
val pairs = Seq(("a",3),("b",4),("b",1),("c",2))
val input = sc.parallelize(pairs)
```

```scala
// Initialization code; we load the user info from a Hadoop SequenceFile on HDFS.
// This distributes elements of userData by the HDFS block where they are found,
// and doesn't provide Spark with any way of knowing in which partition a
// particular UserID is located.
val sc = new SparkContext(...)
val userData = sc.sequenceFile[UserID, UserInfo]("hdfs://...").persist()
// Function called periodically to process a logfile of events in the past 5 minutes;
// we assume that this is a SequenceFile containing (UserID, LinkInfo) pairs.
def processNewLogs(logFileName: String) {
	val events = sc.sequenceFile[UserID, LinkInfo](logFileName)
	val joined = userData.join(events)// RDD of (UserID, (UserInfo, LinkInfo)) pairs
	val offTopicVisits = joined.filter {
		case (userId, (userInfo, linkInfo)) => // Expand the tuple into its components
			!userInfo.topics.contains(linkInfo.topic)
	}.count()
	println("Number of visits to non-subscribed topics: " + offTopicVisits)
}
```

This code will run fine as is, but it will be inefficient. This is because the join() operation, called each time processNewLogs() is invoked, does not know anything about how the keys are partitioned in the datasets. By default, this operation will hash all the keys of both datasets, sending elements with the same key hash across the network to the same machine, and then join together the elements with the same key on that machine. Because we expect the userData table to be much larger than the small log of events seen every five minutes, this wastes a lot of work: the userData table is hashed and shuffled across the network on every call, even though it doesn’t change.

```scala
val sc = new SparkContext(...)
val userData = sc.sequenceFile[UserID, UserInfo]("hdfs://...")
		.partitionBy(new HashPartitioner(100)) // Create 100 partitions
		.persist()
```

The processNewLogs() method can remain unchanged: the events RDD is local to processNewLogs() , and is used only once within this method, so there is no advantage in specifying a partitioner for events . Because we called partitionBy() when building userData , Spark will now know that it is hash-partitioned, and calls to join() on it will take advantage of this information. In particular, when we call user Data.join(events) , Spark will shuffle only the events RDD, sending events with each particular UserID to the machine that contains the corresponding hash partition of userData (see Figure 4-5). The result is that a lot less data is communicated over the network, and the program runs significantly faster.

In fact, many other Spark operations automatically result in an RDD with known partitioning information, and many operations other than join() will take advantage of this information. For example, sortByKey() and groupByKey() will result in range-partitioned and hash-partitioned RDDs, respectively. On the other hand, operations like map() cause the new RDD to forget the parent’s partitioning information, because such operations could theoretically modify the key of each record.

To use 
```scala
val partitioned = pairs.partitionBy(new HashPartitioner(2))
```
You need to import org.apache.spark.HashPartitioner first.

1. Initialize each page’s rank to 1.0.
2. On each iteration, have page p send a contribution of rank(p) / numNeighbors(p)
to its neighbors (the pages it has links to).
3. Set each page’s rank to 0.15 + 0.85 * contributionsReceived .

```scala
// Assume that our neighbor list was saved as a Spark objectFile
val links = sc.objectFile[(String, Seq[String])]("links")
	.partitionBy(new HashPartitioner(100))
	.persist()
// Initialize each page's rank to 1.0; since we use mapValues, the resulting RDD
// will have the same partitioner as links
var ranks = links.mapValues(v => 1.0)
// Run 10 iterations of PageRank
for (i <- 0 until 10) {
	val contributions = links.join(ranks).flatMap {
		case (pageId, (links, rank)) =>
			links.map(dest => (dest, rank / links.size))
	}
	ranks = contributions.reduceByKey((x, y) => x + y).mapValues(v => 0.15 + 0.85*v)
}
// Write out the final ranks
ranks.saveAsTextFile("ranks")
```
1. Notice that the links RDD is joined against ranks on each iteration. Since links is a static dataset, we partition it at the start with partitionBy() , so that it does not need to be shuffled across the network. In practice, the links RDD is also likely to be much larger in terms of bytes than ranks , since it contains a list of neighbors for each page ID instead of just a Double , so this optimization saves considerable network traffic over a simple implementation of PageRank (e.g., in plain MapReduce).
2. For the same reason, we call persist() on links to keep it in RAM across iterations.
3. When we first create ranks , we use mapValues() instead of map() to preserve the partitioning of the parent RDD ( links ), so that our first join against it is cheap.
4. In the loop body, we follow our reduceByKey() with mapValues() ; because the result of reduceByKey() is already hash-partitioned, this will make it more efficient to join the mapped result against links on the next iteration.
