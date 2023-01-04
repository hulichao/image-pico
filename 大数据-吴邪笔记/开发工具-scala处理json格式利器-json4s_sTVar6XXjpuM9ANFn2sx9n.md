# 开发工具-scala处理json格式利器-json4s

# 1.为什么是json4s

从json4s的官方描述

> At this moment there are at least 6 json libraries for scala, not counting the java json libraries. All these libraries have a very similar AST. This project aims to provide a single AST to be used by other scala json libraries.
> At this moment the approach taken to working with the AST has been taken from lift-json and the native package is in fact lift-json but outside of the lift project.

在scala库中，至少有6个json库，并且不包括 java的json库，这些库都有着类似的抽象语法树AST，json4s的目的就是为了使用简单的一种语法支持这些json库，因此说json4s可以说是一种json的规范处理，配合scala开发过程中极其简介的语法特性，可以轻松地实现比如json合并，json的diff操作，可以方便地处理jsonArray的字符串，所以如果使用scala，那么json4s一定不能错过，在实际场景下使用json处理数据很常见，比如spark开发中处理原始json数据等等，开始上手可能看起来比较复杂，但是用起来你会很爽。

# 2.json4s的数据结构

json4s包括10个类型和一个type类型的对象，分别如下

```java
case object JNothing extends JValue // 'zero' for JValue
case object JNull extends JValue
case class JString(s: String) extends JValue
case class JDouble(num: Double) extends JValue
case class JDecimal(num: BigDecimal) extends JValue
case class JInt(num: BigInt) extends JValue
case class JLong(num: Long) extends JValue
case class JBool(value: Boolean) extends JValue
case class JObject(obj: List[JField]) extends JValue
case class JArray(arr: List[JValue]) extends JValue
 
type JField = (String, JValue)
```

可以看到，他们都继承自JValue,JValue是json4s里面类似于java的object地位，而JField是用来一次性匹配json的key,value对而准备的。

# 3.json4s的实践

下面来看，我们如何来使用json4s

```xml
<dependency>
    <groupId>org.json4s</groupId>
    <artifactId>json4s-native_2.11</artifactId>
    <version>3.7.0-M6</version>
</dependency>
```

看下面的代码即可，注释写的比较清晰，一般来说json的使用无外乎是字符串到对象或者对象到字符串，而字符串到对象可以用case class 也可以用原始的比如上面提到的类

```scala
package com.hoult.scala.json4s
import org.json4s._
import org.json4s.JsonDSL._
import org.json4s.native.JsonMethods._

object Demo1 {
  def main(args: Array[String]): Unit = {

    //parse方法表示从字符串到json-object
    val person = parse(
      """
        |{"name":"Toy","price":35.35}
        |""".stripMargin, useBigDecimalForDouble = true)
    // 1.模式匹配提取， \表示提取
    val JString(name) = (person \ "name")
    println(name)

    // 2.extract[String]取值
//    implicit val formats = org.json4s.Formats
    implicit val formats = DefaultFormats
    val name2 = (person \ "name").extract[String]
    val name3 = (person \ "name").extractOpt[String]
    val name4 = (person \ "name").extractOrElse("")

    // 3.多层嵌套取值
    val parseJson: JValue = parse(
      """
        |{"name":{"tome":"new"},"price":35.35}
        |""".stripMargin, useBigDecimalForDouble = true)

    //3.1 逐层访问
    val value = (parseJson \ "name" \ "tome").extract[String]
    //3.2 循环访问
    val value2 = (parseJson \\ "tome")
    println(value2)


    //4.嵌套json串解析
    val json = parse(
      """
         { "name": "joe",
           "children": [
             {
               "name": "Mary",
               "age": 20
             },
             {
               "name": "Mazy",
               "age": 10
             }
           ]
         }
      """)

//    println(json \ "children")

    //模式匹配
    for (JArray(child) <- json) println(child)

    //提取object 下 某字段的值
    val ages = for {
      JObject(child) <- json
      JField("age", JInt(age)) <- child
    } yield age

    println(ages)

    // 嵌套取数组中某个字段值，并添加过滤
    val nameAges = for {
      JObject(child) <- json
      JField("name", JString(name)) <- child
      JField("age", JInt(age)) <- child
      if age > 10
    } yield (name, age)

    println(nameAges)

    // 5.json和对象的转换,[就是json数组]
    case class ClassA(a: Int, b: Int)

    val json2: String = """[{"a":1,"b":2},{"a":1,"b":2}]"""

    val bb: List[ClassA] = parse(json2).extract[List[ClassA]]
    println(bb)

    // 6.json转对象，[json 非json数组，但是每个级别要明确]
    case class ClassC(a: Int, b: Int)

    case class ClassB(c: List[ClassC])

    val json3: String = """{"c":[{"a":1,"b":2},{"a":1,"b":2}]}"""

    val cc: ClassB = parse(json3).extract[ClassB]
    println(cc)

    // 7.使用org.json4s产生json字符串
//    import org.json4s.JsonDSL._
    val json1 = List(1, 2, 3)
    val jsonMap = ("name" -> "joe")
    val jsonUnion = ("name" -> "joe") ~ ("age" -> 10)
    val jsonOpt = ("name" -> "joe") ~ ("age" -> Some(1))
    val jsonOpt2 = ("name" -> "joe") ~ ("age" -> (None: Option[Int]))
    case class Winner(id: Long, numbers: List[Int])
    case class Lotto(id: Long, winningNumbers: List[Int], winners: List[Winner], drawDate: Option[java.util.Date])

    val winners = List(Winner(10, List(1, 2, 5)), Winner(11, List(1, 2, 0)))
    val lotto = Lotto(11, List(1, 2, 5), winners, None)

    val jsonCase =
      ("lotto" ->
        ("lotto-id" -> lotto.id) ~
          ("winning-numbers" -> lotto.winningNumbers) ~
          ("draw-date" -> lotto.drawDate.map(_.toString)) ~
          ("winners" ->
            lotto.winners.map { w =>
              (("winner-id" -> w.id) ~
                ("numbers" -> w.numbers))}))

    println(compact(render(json1)))
    println(compact(render(jsonMap)))
    println(compact(render(jsonUnion)))
    println(compact(render(jsonOpt)))
    println(compact(render(jsonOpt2)))
    println(compact(render(jsonCase)))

    // 8.json格式化
    println(pretty(render(jsonCase)))

    // 9.合并字符串
    val lotto1 = parse("""{
         "lotto":{
           "lotto-id": 1,
           "winning-numbers":[7,8,9],
           "winners":[{
             "winner-id": 1,
             "numbers":[7,8,9]
           }]
         }
       }""")

    val lotto2 = parse("""{
         "lotto":{
           "winners":[{
             "winner-id": 2,
             "numbers":[1,23,5]
           }]
         }
       }""")

    val mergedLotto = lotto1 merge lotto2
//    println(pretty(render(mergedLotto)))

    // 10.字符串寻找差异
    val Diff(changed, added, deleted) = mergedLotto diff lotto1
    println(changed)
    println(added)
    println(deleted)

    val json10 = parse(
      """

      """)

    println("********8")
    println(json10)
    for (JObject(j) <- json10) println(j)

    println("********11")

    // 11.遍历json，使用for
    // key1 values key1_vk1:v1 ....
    val str = "{\"tag_name\":\"t_transaction_again_day\",\"tag_distribute_json\":\"{\\\"1\\\":\\\"0.0011231395\\\",\\\"0\\\":\\\"0.9988768605\\\"}\"}"

    val valueJson = parse(str) \ "tag_distribute_json"
    println(valueJson)
    for {
      JString(obj) <- valueJson
      JObject(dlist) <- parse(obj)
      (key, JString(value))<- dlist
    } {
      println(key + "::" + value)
//      val kvList = for (JObject(key, value) <- parse(obj)) yield (key, value)
//      println("obj : " + kvList.mkString(","))
    }
  }
}

```

# 4.注意

## 4.1 compact 和 render的使用

常用写法`compact(render(json))`,用来把一个json对象转成字符串，并压缩显示，当然也可以用`prety(render(json))`

## 4.2 序列化时候需要一个隐式对象

例如下面的

```scala
implicit val formats = Serialization.formats(NoTypeHints)
```

# 参考

更多参考官网或者非官方博客等

[https://json4s.org/](https://json4s.org/ "https://json4s.org/")

[https://github.com/json4s/json4s/tree/v.3.2.0\_scala2.10](https://github.com/json4s/json4s/tree/v.3.2.0_scala2.10 "https://github.com/json4s/json4s/tree/v.3.2.0_scala2.10")

[https://www.cnblogs.com/yyy-blog/p/11819302.html](https://www.cnblogs.com/yyy-blog/p/11819302.html "https://www.cnblogs.com/yyy-blog/p/11819302.html")

[https://www.shuzhiduo.com/A/Vx5MBVOYdN/](https://www.shuzhiduo.com/A/Vx5MBVOYdN/ "https://www.shuzhiduo.com/A/Vx5MBVOYdN/") &#x20;

[https://segmentfault.com/a/1190000007302496](https://segmentfault.com/a/1190000007302496 "https://segmentfault.com/a/1190000007302496")

[https://www.coder.work/article/6786418](https://www.coder.work/article/6786418 "https://www.coder.work/article/6786418")

[https://www.wolai.com/sTVar6XXjpuM9ANFn2sx9n#xcy85CRuHfRepDBAvtvdq9](https://www.wolai.com/sTVar6XXjpuM9ANFn2sx9n#xcy85CRuHfRepDBAvtvdq9 "https://www.wolai.com/sTVar6XXjpuM9ANFn2sx9n#xcy85CRuHfRepDBAvtvdq9")

[https://www.wolai.com/sTVar6XXjpuM9ANFn2sx9n#7kKK1H1h2GPZnzhiXvWG38](https://www.wolai.com/sTVar6XXjpuM9ANFn2sx9n#7kKK1H1h2GPZnzhiXvWG38 "https://www.wolai.com/sTVar6XXjpuM9ANFn2sx9n#7kKK1H1h2GPZnzhiXvWG38")
