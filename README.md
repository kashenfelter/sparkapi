Spark API Interface
================

<!---
// TODO: sparkapi_dependency
// TODO: sparkapi_dataframe
// TODO: magrittr re-export
--->
Introduction
------------

The [SparkR](https://github.com/apache/spark/tree/master/R) package introduced a custom RPC method for communicating between R and the JVM. This allows for calling arbitrary Java/Scala code from R without the use of the rJava package.

The **sparkapi** package factors out the core RPC protocol from SparkR, providing a mechanism for additional front-end packages (e.g. a dplyr-interface) to use the same protocol as well as extension packages to be written that support multiple front-end packages.

To goal of the **sparkapi** package is to make the core types used by Spark front-end packages interoperable, enabling a constellation of R packages that provide interfaces between R and Spark, all of which work well together.

Core Types
----------

The connection between R client packages and the RBackend is represented by an object of class `sparkapi_connection`. The connection enables creation of new Java objects as well as invoking static methods of Java classes.

An instance of a remote JVM object is represented by an object of class `sparkapi_jobj`. For any given `sparkapi_jobj` it's possible to discover it's underlying `sparkapi_connection`.

Calling Spark from R
--------------------

There are several functions available for calling the methods of Java objects and static methods of Java classes:

| Function              | Description                                    |
|-----------------------|------------------------------------------------|
| sparkapi\_invoke      | Call a method on an object.                    |
| spark\_invoke\_new    | Create a new object by invoking a constructor. |
| spark\_invoke\_static | Call a static method on an object.             |

For example, to create a new instance of the `java.math.BigInteger` class and then call the `longValue()` method on it you would use code like this:

``` r
billionBigInteger <- sparkapi_invoke_new(sc, "java.math.BigInteger", "1000000000")
billion <- sparkapi_invoke(billionBigInteger, "longValue")
```

This code can be re-written to be more compact and clear using [magrittr](https://cran.r-project.org/web/packages/magrittr/vignettes/magrittr.html) pipes:

``` r
billion <- sc %>% 
  sparkapi_invoke_new("java.math.BigInteger", "1000000000") %>%
    sparkapi_invoke("longValue")
```

This syntax is similar to the method-chaining syntax often used with Scala code so is generally preferred.

Calling a static method of a class is also straightforward. For example, to call the `Math::hypot()` static function you would use this code:

``` r
hypot <- sc %>% sparkapi_invoke_static("java.lang.Math", "hypot", 10, 20) 
```

Note that arguments to methods are included immediately after the name of the method to be called.

Wrapper Functions
-----------------

Creating an extension typically consists of writing R wrapper functions for a set of Spark services. In this section we'll describe the typical form of these functions as well as how to handle special types like Spark DataFrames.

Here's an example of a wrapper function for the the text file line counting function available from the SparkContext object:

``` r
count_lines <- function(sc, path) {
  sc %>% sparkapi_invoke("textFile", path, as.integer(1)) %>% 
    sparkapi_invoke("count")
}
```

The `count_lines` function takes a `sparkapi_connection` (`sc`) argument, which when passed to `sparkapi_invoke` enables executing methods of the SparkContext object. Creating new objects also requires the `sc` argument.

In some cases you'll write wrapper functions that accept references to Spark objects (for example, a Spark DataFrame). In this scenario the following functions are also useful:

<table>
<colgroup>
<col width="38%" />
<col width="61%" />
</colgroup>
<thead>
<tr class="header">
<th>Function</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td>sparkapi_connection</td>
<td>Get the Spark connection associated with objects of various types.</td>
</tr>
<tr class="even">
<td>sparkapi_jobj</td>
<td>Get the Spark jobj associated with objects of various types.</td>
</tr>
<tr class="odd">
<td>sparkapi_dataframe</td>
<td>Get a reference to a Spark DataFrame from objects of various types.</td>
</tr>
</tbody>
</table>

The use of these functions is illustrated in this (overly) simple example:

``` r
analyze <- function(x, features) {
  
  # normalize whatever we were passed (e.g. a dplyr tbl) into a Spark DataFrame
  df <- sparkapi_dataframe(x)
  
  # get the underlying connection so we can create new objects
  sc <- sparkapi_connection(df)
  
  # create an object to do the analysis and call its `analyze` and `summary`
  # methods (note that the df and features are passed to the analyze function)
  summary <- sc %>%  
    sparkapi_invoke_new("com.example.tools.Analyzer") %>% 
      sparkapi_invoke("analyze", df, features) %>% 
      sparkapi_invoke("summary")

  # return the results
  summary
}
```

The first argument is an object that can be accessed using the Spark DataFrame API (this might be an actual reference to a DataFrame or could rather be a dplyr `tbl` which has a DataFrame reference inside it). After using the `sparkapi_dataframe` function to normalize the reference, we call `sparkapi_connection` to extract the underlying Spark connection associated with the data frame. Finally, we create a new `Analyzer` object, call it's `analyze` method with the DataFrame and list of features, and then call the `summary` method on the results of the analysis.

Accepting a DataFrame as the first argument of a function makes it very easy to incorporate into magrittr pipelines so this pattern is highly recommended when possible.

Dependencies
------------

When creating R packages which implement interaces to Spark you may need to include additional dependencies. Your dependencies might be a set of [Spark Packages](https://spark-packages.org/) or might be a custom JAR file. In either case, you'll need a way to specify that these dependencies be included during the initialization of a Spark session. A Spark dependency is defined using the `sparkapi_dependency` function:

<table>
<colgroup>
<col width="38%" />
<col width="61%" />
</colgroup>
<thead>
<tr class="header">
<th>Function</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td>sparkapi_dependency</td>
<td>Define a Spark dependency consisting of JAR files and Spark packages.</td>
</tr>
</tbody>
</table>

Your extension package can specify it's dependencies by implementing a function named `sparkapi_dependencies` within the package (this function should *not* be publicly exported). For example, let's say you were creating an extension package named **sparkds** that needed to include a custom JAR as well as the Redshift and Apache Avro packages:

``` r
sparkapi_dependencies <- function(config, ...) {
  sparkapi_dependency(
    jars = system.file("java/sparkds.jar", package = "sparkds"),
    packages = c("com.databricks:spark-redshift_2.10:0.6.0",
                 "com.databricks:spark-avro_2.10:2.0.1")
  )
}
```

The `...` argument is unused but nevertheless should be included to ensure continued compatibility if new arguments are added to `sparkapi_dependencies` in the future.

When users connect to a Spark cluster and want to use your extension package within their session they simply include the **sparkds** package in a list of extensions passed to e.g. `SparkR.init`:

``` r
library(SparkR)
sc <- sparkR.init(master = "local[*]", extensions = c("sparkds"))
```

There are two helper functions which enable front end packages to discover and read Spark dependencies from extension packages as well as convert them into shell arguments suitable for passing to `spark-shell`:

<table>
<colgroup>
<col width="38%" />
<col width="61%" />
</colgroup>
<thead>
<tr class="header">
<th>Function</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td>sparkapi_dependencies_from_extension</td>
<td>Query an extension package for it's list of sparkapi dependencies</td>
</tr>
<tr class="even">
<td>sparkapi_dependencies_to_shellargs</td>
<td>Convert a list of sparkapi dependencies into shell arguments.</td>
</tr>
</tbody>
</table>