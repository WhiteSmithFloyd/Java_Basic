# Java8 Stream

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

```



















