val lines = sc.textFile("/path/to/File")
val sparklines = lines.filter(line => line.contains("spark")
val shelllines = lines.filter(line => line.contains("shell") 
val sparkandshell= sparklines.union(shelllines)
println("Input had " + sparkandshell.count() + " concerning lines")
println("Here are 10 examples:")
sparkandshell.take(10).foreach(println)
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

