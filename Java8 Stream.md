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





















