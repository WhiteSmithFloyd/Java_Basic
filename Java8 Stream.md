# Java8 Stream
From http://blog.csdn.net/u010425776/article/details/52344425
[TOC]

## 什么是流
流是Java8引入的全新概念，它用来处理集合中的数据，暂且可以把它理解为一种高级集合。

### 流的特点
+ 只能遍历一次 
+ 采用内部迭代方式 

### 流的操作种类
+ 中间操作
> 当数据源中的数据上了流水线后，这个过程对数据进行的所有操作都称为“中间操作”。
> 中间操作仍然会返回一个流对象，因此多个中间操作可以串连起来形成一个流水线。
+ 终端操作
> 当所有的中间操作完成后，若要将数据从流水线上拿下来，则需要执行终端操作。
> 终端操作将返回一个执行结果，这就是你想要的数据。


### 流的操作过程
使用流一共需要三步：
1. 准备一个数据源
2. 执行中间操作
  中间操作可以有多个，它们可以串连起来形成流水线。
3. 执行终端操作
  执行终端操作后本次流结束，你将获得一个执行结果。
  
## 流的使用
### 获得流
可以从集合，数组和值直接获得流
```java
// from collection
List<Person> list = new ArrayList<Person>(); 
Stream<Person> stream = list.stream();

// from array
String[] names = {"chaimm","peter","john"};
Stream<String> stream = Arrays.stream(names);

// from string value
Stream<String> stream = Stream.of("chaimm","peter","john");

// from file
try(Stream lines = Files.lines(Paths.get(“文件路径名”),Charset.defaultCharset())){
//可对lines做一些操作
}catch(IOException e){
}
// PS：Java7简化了IO操作，把打开IO操作放在try后的括号中即可省略关闭IO的代码。
```

### 筛选filter
filter函数接收一个Lambda表达式作为参数，该表达式返回boolean，在执行过程中，流将元素逐一输送给filter，并筛选出执行结果为true的元素。 
```java
List<Person> result = list.stream()
                    .filter(Person::isStudent)
                    .collect(toList());
```

### 去重distinct
```java
List<Person> result = list.stream()
                    .distinct()
                    .collect(toList());
```

### 截取
截取流的前N个元素：
```java
List<Person> result = list.stream()
                    .limit(3)   
                    .collect(toList());
```

### 跳过
跳过流的前n个元素：
```java
List<Person> result = list.stream()
                    .skip(3)
                    .collect(toList());
```

### 映射
对流中的每个元素执行一个函数，使得元素转换成另一种类型输出。流会将每一个元素输送给map函数，并执行map中的Lambda表达式，最后将执行结果存入一个新的流中。 
```java
List<Person> result = list.stream()
                    .map(Person::getName)
                    .collect(toList());
```

### 合并多个流
```java
List<String> list = new ArrayList<String>();
list.add("I am a boy");
list.add("I love the girl");
list.add("But the girl loves another girl");

// to stream
list.stream()
   .map(line->line.split(" "))    // 安空格分词
   // 此时一个大流里面包含了一个个小流
   .map(Arrays::stream)           // 将每个String[]变成流
   .flagmap(Arrays::stream)       // 将小流合并成一个大流
   .distinct()                    // 去重
   .collect(toList());

```

### 是否匹配任一元素：anyMatch
anyMatch用于判断流中是否 __存在至少一个元素__ 满足指定的条件，这个判断条件通过Lambda表达式传递给anyMatch，执行结果为boolean类型。

### 是否匹配所有元素：allMatch
allMatch用于判断流中是否 __所有元素都满足__ 指定条件，这个判断条件通过Lambda表达式传递给anyMatch，执行结果为boolean类型。 

### 是否未匹配所有元素：noneMatch
noneMatch与allMatch恰恰相反，它用于判断流中的 __所有元素是否都不满足__ 指定条件

### 获取任一元素findAny
findAny能够从流中随便选一个元素出来，它返回一个 __Optional类型__ 的元素
```java
Optional<Person> person = list.stream()
                                    .findAny();
```

#### Optional介绍
Optional是Java8新加入的一个容器，这个容器 __只存1个或0个元素__ ，它用于防止出现NullpointException，它提供如下方法：
> __isPresent()__   
>   判断容器中是否有值。   
> __ifPresent(Consume lambda)__   
>   容器若不为空则执行括号中的Lambda表达式。   
> __T get()__   
>   获取容器中的元素，若容器为空则抛出NoSuchElement异常。   
> __T orElse(T other)__   
>   获取容器中的元素，若容器为空则返回括号中的默认值。   

### 获取第一个元素findFirst
```java
Optional<Person> person = list.stream()
                                    .findFirst();
```

### 归约
归约是将集合中的所有元素经过指定运算，折叠成一个元素输出，如：求最值、平均数等，这些操作都是将一个集合的元素折叠成一个元素输出。   
在流中，reduce函数能实现归约。 
reduce函数接收两个参数：
+ 初始值
+ 进行归约操作的Lambda表达式
   
   
#### 元素求和 (自定义Lambda表达式实现求和)
计算所有人的年龄总和
```java
int age = list.stream().reduce(0, (person1,person2)->person1.getAge()+person2.getAge());
```
reduce的第一个参数表示初试值为0；    
reduce的第二个参数为需要进行的归约操作，它接收一个拥有两个参数的Lambda表达式，reduce会把流中的元素__两两__输给Lambda表达式，最后将计算出累加之和。    

#### 元素求和 (使用Integer.sum函数求和)
如果当前流的元素为数值类型，那么可以使用Integer提供了sum函数代替自定义的Lambda表达式，如：
```java
int age = list.stream().reduce(0, Integer::sum);
```
Integer类还提供了min、max等一系列数值操作，当流中元素为数值类型时可以直接使用。   
   
   

### 数值流的使用
采用reduce进行数值操作会涉及到基本数值类型和引用数值类型之间的装箱、拆箱操作，因此效率较低。    
当流操作为纯数值操作时，使用数值流能获得较高的效率。
   
   
#### 将普通流转换成数值流
StreamAPI提供了三种数值流：IntStream、DoubleStream、LongStream，也提供了将普通流转换成数值流的三种方法：mapToInt、mapToDouble、mapToLong。   
```java
IntStream stream = list.stream()
                            .mapToInt(Person::getAge);
```
   
   
#### 数值计算
每种数值流都提供了数值计算函数，如max、min、sum等。   
如，找出最大的年龄：   
```java
OptionalInt maxAge = list.stream()
                                .mapToInt(Person::getAge)
                                .max();
/* 由于数值流可能为空，并且给空的数值流计算最大值是没有意义的，因此max函数返回OptionalInt
 * 它是Optional的一个子类，能够判断流是否为空，并对流为空的情况作相应的处理。                                
 */
```
此外，mapToInt、mapToDouble、mapToLong进行数值操作后的返回结果分别为：OptionalInt、OptionalDouble、OptionalLong

   
   
## 收集器简介
收集器用来将经过筛选、映射的流进行最后的整理，可以使得最后的结果以不同的形式展现。   

**collect方法即为收集器**，它接收**Collector接口的实现**作为具体收集器的收集方法。
> Collector接口提供了很多默认实现的方法，我们可以直接使用它们格式化流的结果；   
> 也可以自定义Collector接口的实现，从而定制自己的收集器。

下面介绍Collector常用默认静态方法的使用

   
      
   
## 收集器的使用   
### 归约
流由一个个元素组成，归约就是将一个个元素“折叠”成一个值，如求和、求最值、求平均值都是归约操作   
   
      
   
#### 计数
```java
long count = list.stream()
      .collect(Collectors.counting());
```
也可以不使用收集器的计数函数：
```java
long count = list.stream().count();
```
   
      
   
#### 最值
找出所有人中年龄最大的人
```java
Optional<Person> oldPerson = list.stream()
      .collect(Collectors.maxBy(Comparator.comparingInt(Person::getAge)));
```
计算最值需要使用Collector.maxBy和Collector.minBy，这两个函数需要传入一个比较器**Comparator.comparingInt**，这个比较器又要接收需要比较的字段。

   
   
#### 求和
计算所有人的年龄总和
```java
int summing = list.stream()
      .collect(Collectors.summingInt(Person::getAge));
```
既然Java8提供了summingInt，那么还提供了summingLong、summingDouble。

   
   
#### 求平均值
计算所有人的年龄平均值
```java
double avg = list.stream()
      .collect(Collectors.averagingInt(Person::getAge));
```
计算平均值时，不论计算对象是int、long、double，计算结果一定都是double。



#### 一次性计算所有归约操作
**Collectors.summarizingInt**函数能一次性将最值、均值、总和、元素个数全部计算出来，并存储在对象IntSummaryStatisics中。    
可以通过该对象的getXXX()函数获取这些值。
   
   

#### 连接字符串
将所有人的名字连接成一个字符串
```java
String names = list.stream()
      .collect(Collectors.joining());
```
每个字符串默认分隔符为空格，若需要指定分隔符，则在joining中加入参数即可：
```java
String names = list.stream()
      .collect(Collectors.joining(", "));
```
此时字符串之间的分隔符为逗号。




#### 一般性的归约操作
若你需要自定义一个归约操作，那么需要使用Collectors.reducing函数，该函数接收三个参数：
+ 第一个参数为归约的初始值
+ 第二个参数为归约操作进行的字段
+ 第三个参数为归约操作的过程

计算所有人的年龄总和
```java
Optional<Integer> sumAge = list.stream()
      .collect(Collectors.reducing(0,Person::getAge,(i,j)->i+j));
```
上面例子中，reducing函数一共接收了三个参数：
+ 第一个参数表示归约的初始值。我们需要累加，因此初始值为0
+ 第二个参数表示需要进行归约操作的字段。这里我们对Person对象的age字段进行累加。
+ 第三个参数表示归约的过程。这个参数接收一个Lambda表达式，而且这个Lambda表达式一定拥有两个参数，分别表示当前相邻的两个元素。由于我们需要累加，因此我们只需将相邻的两个元素加起来即可。

**Collectors.reducing方法**还提供了一个单参数的重载形式。 
你只需传一个归约的操作过程给该方法即可(即第三个参数)，其他两个参数均使用默认值。
+ 第一个参数默认为流的第一个元素
+ 第二个参数默认为流的元素 
这就意味着，当前流的元素类型为数值类型，并且是你要进行归约的对象。    
*采用单参数的reducing计算所有人的年龄总和*
```java
Optional<Integer> sumAge = list.stream()
            .filter(Person::getAge)
            .collect(Collectors.reducing((i,j)->i+j));
```



### 分组
分组就是将流中的元素按照指定类别进行划分，类似于SQL语句中的GROUPBY。


#### 一级分组
将所有人分为老年人、中年人、青年人
```java
Map<String,List<Person>> result = list.stream()
        .collect(Collectors.groupingby( (person)->{
          if(person.getAge()>60)
              return "老年人";
          else if(person.getAge()>40)
              return "中年人";
          else
              return "青年人";
          }
        )
);
```
groupingby函数接收一个Lambda表达式，该表达式返回String类型的字符串，groupingby会**将当前流中的元素按照Lambda返回的字符串进行分组**。 


#### 多级分组
多级分组可以支持在完成一次分组后，分别对每个小组再进行分组。 
使用具有两个参数的groupingby重载方法即可实现多级分组。
+ 第一个参数：一级分组的条件
+ 第二个参数：一个新的groupingby函数，该函数包含二级分组的条件

*将所有人分为老年人、中年人、青年人，并且将每个小组再分成：男女两组。*
```java
Map<String,Map<String,List<Person>>> result = list.stream()
      .collect(Collectors.groupingby(
            (person)->{
           if(person.getAge()>60)
                return "老年人";
           else if(person.getAge()>40)
                return "中年人";
           else
               return "青年人";
           },
            groupingby(Person::getSex)));
```
此时会返回一个非常复杂的结果：Map<String, Map<String, List<Person>>>。


#### 对分组进行统计
拥有两个参数的groupingby函数不仅仅能够实现多几分组，还能对分组的结果进行统计。
*统计每一组的人数*
```java
Map<String,Long> result = list.stream()
      .collect(Collectors.groupingby((person)->{
          if(person.getAge()>60)
              return "老年人";
         else if(person.getAge()>40)
             return "中年人";
         else
              return "青年人";
         },
         counting()));
```
此时会返回一个Map<String,Long>类型的map，该map的键为组名，map的值为该组的元素个数。

##### 将收集器的结果转换成另一种类型
当使用maxBy、minBy统计最值时，结果会封装在Optional中，该类型是为了避免流为空时计算的结果也为空的情况。在单独使用maxBy、minBy函数时确实需要返回Optional类型，这样能确保没有空指针异常。然而当我们使用groupingBy进行分组时，若一个组为空，则该组将不会被添加到Map中，从而Map中的所有值都不会是一个空集合。既然这样，使用maxBy、minBy方法计算每一组的最值时，将结果封装在optional对象中就显得有些多余。      

我们可以使用collectingAndThen函数包裹maxBy、minBy，从而将maxBy、minBy返回的Optional对象进行转换。

*例：将所有人按性别划分，并计算每组最大的年龄。*
```java
Map<String,Integer> map = list.stream()
  .collect(groupingBy(Person::getSex,
        collectingAndThen(
            maxBy(comparingInt(Person::getAge)),
        Optional::get
)));
```
此时返回的是一个Map< String,Integer>，String表示每组的组名(男、女)，Integer为每组最大的年龄。 
如果不用collectingAndThen包裹maxBy，那么最后返回的结果为Map< String,Optional< Person>>。 
使用collectingAndThen包裹maxBy后，首先会执行maxBy函数，该函数执行完后便会执行Optional::get，从而将Optional中的元素取出来。


#### 分区
分区是分组的一种特殊情况，它只能分成true、false两组。 
分组使用partitioningBy方法，该方法接收一个Lambda表达式，该表达是必须返回boolean类型，partitioningBy方法会将Lambda返回结果为true和false的元素各分成一组。 
partitioningBy方法返回的结果为Map< Boolean,List< T>>。 
此外，partitioningBy方法和groupingBy方法一样，也可以接收第二个参数，实现二级分区或对分区结果进行统计。



