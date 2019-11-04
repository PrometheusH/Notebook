### Scala基本语法

初始化Seq：

```scala
var accumUpdates: Seq[AccumulatorV2[_, _]] = Seq.empty
```

声明参数：

```scala
private var _applicationId: String = _
```

说明：

```scala
/* ------------------------------------------------------------------------------------ *
   | Private variables. These variables keep the internal state of the context, and are |
   | not accessible by the outside world. They're mutable since we want to initialize al|
   | of them to some neutral value ahead of time, so that calling "stop()" while the    |
   | constructor is still running is safe.                                              |
   * --------------------------------------------------------------------------------- */
```

如何使用正则表达式：

```scala
private object SparkMasterRegex {
    // Regular expression used for local[N] and local[*] master formats
    val LOCAL_N_REGEX = """local\[([0-9]+|\*)\]""".r
    // Regular expression for local[N, maxRetries], used in tests with failing tasks
    val LOCAL_N_FAILURES_REGEX = """local\[([0-9]+|\*)\s*,\s*([0-9]+)\]""".r
    // Regular expression for simulating a Spark cluster of [N, cores, memory] locally
    val LOCAL_CLUSTER_REGEX = """local-cluster\[\s*([0-9]+)\s*,\s*([0-9]+)\s*,\s*([0-9]+)\s*]""".r
    // Regular expression for connecting to Spark deploy clusters
    val SPARK_REGEX = """spark://(.*)""".r
  }

  def main(args: Array[String]): Unit = {
    val master = "local[10,29]";
    master match {
      case "local" => printf("local")
//      case SparkMasterRegex.LOCAL_N_REGEX(threads) => printf("LOCAL_N_REGEX，threads的值是："+threads)
        //括号里放的是第一个正则表达式匹配出的值和第二个正则表达式匹配出的值，后面可以用它们
      // 如果只用其中一个，另一个可以用_表示，但是master不能是"local[10,]"，这样根本就不会进来
      case SparkMasterRegex.LOCAL_N_FAILURES_REGEX(first,second) => printf(s"LOCAL_N_FAILURES_REGEX，第一个值是：$first，第二个值是$second")
      case "local[1]" => printf("local[1]")
      case _ => printf("没有匹配到") // driver is not used for execution
    }
  }
```





























