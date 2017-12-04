# Java8 Stream
[toc]

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
anyMatch用于判断流中是否__存在至少一个元素__满足指定的条件，这个判断条件通过Lambda表达式传递给anyMatch，执行结果为boolean类型。

### 是否匹配所有元素：allMatch
allMatch用于判断流中是否__所有元素都满足__指定条件，这个判断条件通过Lambda表达式传递给anyMatch，执行结果为boolean类型。 

### 是否未匹配所有元素：noneMatch
noneMatch与allMatch恰恰相反，它用于判断流中的__所有元素是否都不满足__指定条件

### 获取任一元素findAny
findAny能够从流中随便选一个元素出来，它返回一个__Optional类型__的元素
```java
Optional<Person> person = list.stream()
                                    .findAny();
```

#### Optional介绍
Optional是Java8新加入的一个容器，这个容器__只存1个或0个元素__，它用于防止出现NullpointException，它提供如下方法：
> __isPresent()__
> 判断容器中是否有值。
> __ifPresent(Consume lambda)__
> 容器若不为空则执行括号中的Lambda表达式。
> __T get()__
> 获取容器中的元素，若容器为空则抛出NoSuchElement异常。
> __T orElse(T other)__
> 获取容器中的元素，若容器为空则返回括号中的默认值。















