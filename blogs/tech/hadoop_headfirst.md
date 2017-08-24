## Hadoop安装（Ubuntu14.04）

1. 安装java
2. 配置JAVA_HOME
3. 下载hadoop2.7解压即可使用


## HDFS

在下面的配置文件中配置HDFS:

etc/hadoop/core-site.xml:

```xml
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://localhost:9000</value>
    </property>
</configuration>
```

etc/hadoop/hdfs-site.xml:

```xml
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
</configuration>
```
 
完成以上两个配置文件尝试进行ssh链接


```shell
$ ssh localhost
```

使用下面的命令进行无密码ssh链接:

```shell
$ ssh-keygen -t dsa -P '' -f ~/.ssh/id_dsa
$ cat ~/.ssh/id_dsa.pub >> ~/.ssh/authorized_keys
$ chmod 0600 ~/.ssh/authorized_keys
```

1. Format the filesystem:

```shell
$bin/hdfs namenode -format
```

2. Start NameNode daemon and DataNode daemon:

```shell
$ sbin/start-dfs.sh
```
 
3. 在浏览器中查看NameNode
NameNode - http://localhost:50070/

4. 创建目录

```shell
$ bin/hdfs dfs -mkdir /user
$ bin/hdfs dfs -mkdir /user/<username>
```

5. 将文件拷贝到HDFS文件系统:

```shell
$ bin/hdfs dfs -put etc/hadoop input
```

6. Run some of the examples provided:

```shell
$ bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.2.jar grep input output 'dfs[a-z.]+'
```

7. Examine the output files: Copy the output files from the distributed filesystem to the local filesystem and examine them:

```shell
$ bin/hdfs dfs -get output output
$ cat output/*
```
  
8. When you’re done, stop the daemons with:
 
```shell
$ sbin/stop-dfs.sh
```
  
## MapReduce
  
编写map reduce文件
  
```java
import java.io.IOException;
import java.util.StringTokenizer;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

public class WordCount {

  public static class TokenizerMapper extends Mapper<Object, Text, Text, IntWritable>{

    private final static IntWritable one = new IntWritable(1);
    private Text word = new Text();

    public void map(Object key, Text value, Context context ) throws IOException, InterruptedException {
      StringTokenizer itr = new StringTokenizer(value.toString());
      while (itr.hasMoreTokens()) {
        word.set(itr.nextToken());
        context.write(word, one);
      }
    }
  }

  public static class IntSumReducer extends Reducer<Text,IntWritable,Text,IntWritable> {
    private IntWritable result = new IntWritable();

    public void reduce(Text key, Iterable<IntWritable> values,Context context) throws IOException, InterruptedException {
      int sum = 0;
      for (IntWritable val : values) {
        sum += val.get();
      }
      result.set(sum);
      context.write(key, result);
    }
  }

  public static void main(String[] args) throws Exception {
    Configuration conf = new Configuration();
    Job job = Job.getInstance(conf, "word count");
    job.setJarByClass(WordCount.class);
    job.setMapperClass(TokenizerMapper.class);
    job.setCombinerClass(IntSumReducer.class);
    job.setReducerClass(IntSumReducer.class);
    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(IntWritable.class);
    FileInputFormat.addInputPath(job, new Path(args[0]));
    FileOutputFormat.setOutputPath(job, new Path(args[1]));
    System.exit(job.waitForCompletion(true) ? 0 : 1);
  }
}
```
 
制作成jar包
 
```shell
$ bin/hadoop com.sun.tools.javac.Main WordCount.java
$ jar cf wc.jar WordCount*.class
```

启动hdfs并将文件提交到hdfs中:参考HDFS

执行程序：

```shell
bin/hadoop jar wc.jar WordCount /user/liangz14/input /user/liangz14/output
```

查看结果

```shell
bin/hadoop fs -cat /user/joe/wordcount/output/part-r-00000`
```
-libjars, -files and -archives: 参数的使用  分别用来制定jar包，指定文件，和未解压文件 通过#也可以致命具体解压后的文件名
，不同参数之间用‘’，‘分割

### mr的一般过程

```xml
(input) <k1, v1> -> map -> <k2, v2> -> combine -> <k2, v2> -> reduce -> <k3, v3> (output)
combine和reducer的作用一样 设置的方法：  job.setCombinerClass(IntSumReducer.class);
```

###  MAIN方法中的设置
#### Mapper ：
通过Job.setMapperClass(Class) 设置map 

map的结果通过context.write(WritableComparable, Writable).进行收集

之后会根据比较器进行分组： Job.setGroupingComparatorClass(Class).

之后map的结果救回被排序 并被划分给不同的reducer，划分任务的个数和reduce任务的个数一样

可以制定那个key进入那个Reducer：通过实现客户自定义的划分器

用户可以选择是有使用一个结合器 Job.setCombinerClass(Class)，来实现中间结果的本地聚合，可以减少数据传输的总量（从Mapper到Reducer）

中间结果总是会进行排序，根据：key的长度，key，值的长度，值。可以控制是否压缩，如何压缩。

#### Maps的数量：
有输入数据的总数决定。输入文件的总的块数

maps的数量最好为每个节点10到100  经过设置到了每个cpu最多300

Configuration.set(MRJobConfig.NUM_MAPS, int) 可以进行设置
 
#### Mapper细节：

Map首先调用setup（context）操作

然后调用map（object，object，context）操作

最后调用cleanup（context）操作

map可以通过RawComparator 控制分组和排序

通过Partitioner控制那个key进入那个Reducer

通过setCombinerClass 可以进行本地的组合

通过 CompressionCodec控制压缩

#### Reducer：
Reducer将中间结果（有共同的key）装换成一个更小的值的集合
 
Job.setNumReduceTasks(int).来设置reduce的数量
 
Job.setReducerClass(Class) 来对reduce进行设置

每一个 \<key, (list of values)\>都会调用一次

Reducer有三个主要的阶段shuffle, sort and reduce.

#### Shuffle：
框架从所有Mapper的排序结果中获取相关的划分

Sort：框架通过key将Reducer的输入进行分组，因为不同的mapper可能有相同key的输出   

Job.setSortComparatorClass(Class).可以生命子集的规则，指定如何分组

Reduce：每一个key都会执行一个reduce，这个结果会通过Context.write写入到文件系统中。

Reduce的输出结果是没有经过排序的

Recude的个数 0.95 *  节点数 * 节点最大容器数 - 1.75 * ...

#### Partitioner
用来将key空间进行划分，最终划分的总数和reduce任务的总数是一样的
HashPartitioner是默认的划分器
 
#### Counter
计数器时统计工具，M R都能用来统计
 
Hadoop MapReduce comes bundled with a library of generally useful mappers, reducers, and partitioners.

Hadoop的MR 是由一系列的 M  R 和P操作库 组成的

#### JOB设置
Mapper, combiner (if any), Partitioner, Reducer, InputFormat, OutputFormat implementations

其中FileInputFormat设置输入文件的集合路径

FileOutputFormat设置输出路径

## HDFS常用命令
  
```shell
hadoop fs -ls /
hadoop fs -lsr
hadoop fs -mkdir /user/hadoop
hadoop fs -put a.txt /user/hadoop/
hadoop fs -get /user/hadoop/a.txt /
hadoop fs -cp src dst
hadoop fs -mv src dst
hadoop fs -cat /user/hadoop/a.txt
hadoop fs -rm /user/hadoop/a.txt
hadoop fs -rmr /user/hadoop/a.txt
hadoop fs -text /user/hadoop/a.txt
hadoop fs -copyFromLocal localsrc dst 与hadoop fs -put功能类似。
hadoop fs -moveFromLocal localsrc dst 将本地文件上传到hdfs，同时删除本地文件。

hadoop dfsadmin -report
hadoop dfsadmin -safemode enter | leave | get | wait
hadoop dfsadmin -setBalancerBandwidth 1000
```

[back](../../index.md)

  
  
