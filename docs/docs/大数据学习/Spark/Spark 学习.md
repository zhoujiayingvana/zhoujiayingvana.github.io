# 基本概念
RDD： 弹性分布式数据集（resilient distributed dataset），df是rdd的一个实现

RDD来源：读取数据集，或者在spark中创建

RDD操作：转换transformations and 行动actions  

1. transformation：转换，中间过程，返回新的RDD，后面可以继续接链式调用。类比Java的Stream API
    1.  pythonLines = lines.filter(lambda line: "Python" in line)  
    2. 转换操作可以操作2个以上的元素
2. action：行动，返回计算结果，后面无法继续链式调用
    1.  pythonLines.first()

RDD只会在action阶段实际调用，一是节约内存，二是可以对完整调用链做优化

RDD默认不存储中间结果，需要显式调用RDD.persist()（默认存储级别下，RDD.cache()与之等价）缓存。

RDD编程步骤：

1. 创建RDD
2. 转换
3. 持久化部分结果
4. 触发计算获得结果

## Python传参注意事项
python传参传函数。经典案例，lambda表达式

```python
word = rdd.filter(lambda s: "error" in s) 
# 等价于
def containsError(s): 
    return "error" in s 
word = rdd.filter(containsError)
```

 传递函数时需要小心的一点是，Python会在你不经意间把函数所在的对象也序列化传出 去。当你传递的对象是某个对象的成员，或者包含了对某个对象中一个字段的引用时（例 如self.field），Spark 就会把整个对象发到工作节点上，这可能比你想传递的东西大得多 （见例3-19）。有时，如果传递的类里面包含Python不知道如何序列化传输的对象，也会 导致你的程序失败  

```python
class SearchFunctions(object): 
  def __init__(self, query): 
      self.query = query 
  def isMatch(self, s): 
      return self.query in s 
  def getMatchesFunctionReference(self, rdd): 
      # 问题：在"self.isMatch"中引用了整个self 
      return rdd.filter(self.isMatch) 
  def getMatchesMemberReference(self, rdd): 
      # 问题：在"self.query"中引用了整个self 
      return rdd.filter(lambda x: self.query in x)
```

 解决方案：只把你所需要的字段从对象中拿出来放到一个**局部变量**中，然后传递这个局部变量

```python
class WordFunctions(object): 
  ... 
  def getMatchesNoReference(self, rdd): 
      # 安全：只把需要的字段提取到局部变量中 
      query = self.query 
      return rdd.filter(lambda x: query in x)
```

  

