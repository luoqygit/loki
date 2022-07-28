# LogQL - 从小白到入门

<br>  

- [前言](#前言)
- [第一个LogQL查询](#第一个logql查询)
- [根据日志内容进行查询](#根据日志内容进行查询)
- [对日志进行统计分析](#对日志进行统计分析)
- [小结](#小结)
- [参考资料](#参考资料)

<br>  


## **前言**  
近期测试了loki和grafana这套日志监控方案，也对loki的日志查询语言LogQL有了一些了解，因此做个记录，免得时间长了自己忘记了。
  
在使用LogQL的时候，最好对kubernetes以及kubernetes标签的概念有一定了解。 

loki的安装配置可以参考我另外一篇文章：[在Minikube上配置Loki日志监控](./README.md)  
  
<br>

## **第一个LogQL查询**  
LogQL的基础是   ***日志流选择器 （log stream selector）*** 。所谓日志流选择器，简单来说就是一对大括号里包含条件表达式，根据表达式来选出满足条件的日志。下面是一个简单的日志流选择器，用来查询loki自身产生的日志：
```
{app="loki"}
```
其中的app是日志的标签名称，这些标签在使用promtail和fluent bit等工具采集日志时，可以自动从kubernetes获取并添加到日志中。  

在查询条件中可以组合多个标签，例如下面的查询，选择app为loki，并且pod名称为loki-0的日志：

```
{app="loki", pod="loki-0"}
```
日志流选择器里可以使用下列运算符：  
| 运算符 | 含义 |
|----|----|
| = | 相等 |
| != | 不相等 |
| =~ | 正则表达式匹配 |
| !~ |正则表达式不匹配|
| >或>= | 大于，或大于等于 |
| <或<= | 小于，或小于等于 |

比如：
```
# 除loki以外其它应用的日志
{app!="loki"}

# grafana或者loki的日志
{app=~"grafana|loki"}
```

<br>  

## **根据日志内容进行查询**  

### **匹配日志内容**
按照标签选出日志以后，可以使用 ***过滤器*** 根据日志内容做进一步筛选，比如下面的例子，查询loki日志且日志中包含error这几个字符：
```
{app="loki"} |= "error"
```

过滤器的语法是：  
***日志流选择器 + 运算符 + 字符串/正则表达式***  

<br>  

在本例里，大括号及括号里的内容是日志流选择器；"|="是运算符，表示包含；"error"是日志中包含的字符串。

日志流选择器如上节所述。
运算符包含下面几种：  
| 运算符 | 含义 |
|---|---|
| \|=  | 日志中包含字符串 |
| \!= | 日志中不包含字符串 |
| \|~  | 日志中包含匹配正则表达式的字符串 |
| \!~ | 日志中不包含匹配正则表达式的字符串 |
  
<br>  

更多的例子：  
```
# 查询loki日志，且日志内容中包含10.24.0这个ip地址段
{app="loki"} |~ "10.24.0.[0-9]+"

# 查询日志内容中包含"level=xxx"的日志
{app="loki"} |~ `level=\w+`
```

过滤器可以级联，就是在过滤器后面再加过滤器，比如：
```
# 由loki生成、包含"error"字符串、且包含10.24.0IP地址段的日志
{app="loki"} |= "error" |~ "10.24.0.[0-9]+"
```
*注: LogQL里有专门对IP地址进行匹配的函数，可以实现更加强大和复杂的IP筛选功能，有兴趣的读者可以去查阅官方文档。这里只是简单举例。*  

  
<br>  

### **提取日志字段并按照字段取值进行查询**
在日志管理场景中，一个常见的需求是提取日志字段并根据这些字段的内容对日志进行查询。比如下面这行日志是json格式，我们可以提取这些json字段，并按照字段值对日志进行过滤。
```
{"log":"logger=context traceID=00000000000000000000000000000000 userId=0 orgId=0 uname= t=2022-07-28T02:21:36.28181343Z level=info msg=\"Request Completed\" method=GET path=/ status=302 remote_addr=10.244.0.9 time_ms=1 duration=1.658433ms size=29 referer= traceID=00000000000000000000000000000000\n","stream":"stdout","time":"2022-07-28T02:21:36.282457104Z"}
```
使用下面的json过滤器解析日志的内容，并把解析出来的字段作为查询条件：
```
# 解析json格式的日志，并把日志里的log, stream, time这三个字段加到日志标签里。
{app="grafana"} | json

# 利用解析出来的标签作为查询条件，只选出stream标签值为stdout的日志
{app="grafana"} | json | stream="stdout"
```
如果更进一步，可以看到log字段本身其实也包含了键/值组合，如果我们想把这些键/值组合也作为查询条件，应该怎么做呢？  
  
这可以通过 **“line format表达式”** 实现，看下面的例子：
```
{app="grafana"} | json | line_format "{{.log}}" | logfmt
```
**json** ：按照json格式对日志内容进行解析，把解析结果加入到日志标签。  
**line_format "{{.log}}"** ：用上一步解析出来的log字段的值替换日志内容。  
**logfmt** ：对日志内容（也就是原来log字段的值）按照logfmt格式进行解析，并把解析出来的键/值加入到日志标签。
<br>  
整个处理过程是一个流水线，每个竖线分割开前后两个处理步骤，前一步的输出作为后一步的输入。
<br>  
经过上面的处理之后，日志添加了status, duration等解析出来的数据作为标签。后续就可以利用这些标签进行查询了：
```
# 查询耗时100ms以上的日志
{app="grafana"} | json | line_format "{{.log}}" | logfmt | duration>100ms
```
<br>  

初看上去，LogQL查询日志内容似乎很繁琐，需要经过多个步骤才能实现，不如elasticsearch等使用的预先解析日志字段的方式直截了当。但是，**个人觉得这是LogQL最大的亮点，这种设计带来极大的灵活性和强大的查询能力。日志后台的采集和处理流程无需改变，就能够适应各种日志格式和内容**。日志格式和字段名称、数量等可以随意变化，只需要在查询的时候设置相应的解析器和查询条件就可以了。

<br>  



## **对日志进行统计分析**
对于查询出来的日志，可以使用LogQL函数进行统计分析。比如：
```
# 以分钟为单位统计日志数量，统计结果会按照标签值的组合进行分组
count_over_time({app="loki"}[1m])
```
上面方括号里的1m表示统计的时间范围是1分钟。也可以是秒或者其它单位，比如15s表示15秒，1h表示1小时。  

Grafana里定义了一些全局的时间范围变量，可以在查询中使用：  
$__interval 表示缺省的时间范围，通常是15s。   
$__range 是在grafana界面上方选择的时间范围。   


因此，可以把上面查询里的时间范围替换成$__interval变量：  
```
count_over_time({app="loki"}[$__interval])
```
<br>  

在实际应用中，通常还会对统计结果进行汇总：  
```
# 按分钟统计日志总条数
sum(count_over_time({app=~".+"}[1m]))

# 汇总统计的时候按照name_space标签进行分组
sum by (name_space) (count_over_time({app=~".+"}[1m]))

# 统计grafana产生的且耗时100ms以上的日志数量，时间间隔1分钟
sum(count_over_time({app="grafana"} | json | line_format "{{.log}}" | logfmt | duration>100ms [1m]))

# 统计每分钟产生的错误日志条数
sum(count_over_time({app=~".+"} | json | line_format "{{.log}}" | logfmt | level="error" [1m]))

# 按照不同级别统计每分钟产生的日志条数
# 统计的时候去掉解析出错的日志：__error__="" 
# 如果解析出现错误，系统会把__error__标签的值设置为错误原因，所以标签值不为空
sum by (level) (count_over_time({app=~".+"} | json | line_format "{{.log}}" | logfmt | __error__="" [1m]))
```

<br>  

## **小结**
LogQL是一种灵活的日志查询语言，和PromQL类似。如果有编写正则表达式的基础，可以很快上手写出各种强大的查询功能。  

<br>  

## **参考资料**  
- [LogQL官方文档](https://grafana.com/docs/loki/latest/logql/)  
- [正则表达式语法](https://github.com/google/re2/wiki/Syntax)
- [五分钟了解LogQL](https://zhuanlan.zhihu.com/p/267222513)