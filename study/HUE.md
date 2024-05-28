# HUE

# 什么是HUE？

![image-20200626094805815](HUE.assets\image-20200626094805815.png)

 	==HUE=HadoopUser Experience==，是一个==开源==的Apache Hadoop UI系统，由Cloudera Desktop演化而来，最后Cloudera公司将其贡献给Apache基金会的Hadoop社区。通过HUE我们可以在浏览器端完成与Hadoop集群的交互工作。



3.11主要特点：

* Hive，Impala，MySQL，Oracle，PostGresl，SparkSQL，Solr SQL，Phoenix的SQL编辑器…
* 使用Solr的动态搜索仪表板
* Spark和Hadoop笔记本
* 通过Oozie编辑器和仪表板安排作业和工作流程

# 使用

## 布局

![image-20200626095034437](HUE.assets/image-20200626095034437.png)

## 功能选择

<img src="HUE.assets/image-20200626110256560.png" alt="image-20200626110256560" style="zoom: 67%;" />

## 文件系统

HDFS文件系统的web界面，类似于NameNode的Browser，但是功能要多一点，更方便使用。

![image-20200626105817014](HUE.assets/image-20200626105817014.png)

![image-20200626110017961](HUE.assets/image-20200626110017961.png)

## HIVE使用

### 控制板结构

左侧为数据库和表的信息，右侧为sql编辑器

![img](https://cdn.gethue.com/uploads/2016/08/autocomp_columns.gif)

### 表的Schema

PS：3.11版本是view more查看schema详细

<img src="HUE.assets/image-20200626112402498.png" alt="image-20200626112402498" style="zoom:67%;" />

![image-20200626112700968](HUE.assets/image-20200626112700968.png)

### 自动补全和智能提示

![img](https://cdn.gethue.com/uploads/2016/08/autocomp_keywords.gif)

![img](https://cdn.gethue.com/uploads/2016/08/autocomp_udf.gif)

### SQL的格式化和Explain

![image-20200626113222641](HUE.assets/image-20200626113222641.png)

explain分析sql执行计划，stage越少，sql的效率越高。

![image-20200626113249365](HUE.assets/image-20200626113249365.png)

### 查询结果导出

![image-20200626113535704](HUE.assets/image-20200626113535704.png)

也可以直接将结果用仪表板展示：

![image-20200626113630980](HUE.assets/image-20200626113630980.png)

## 制图（仪表板）

可以通过配置数据源来做数据的可视化展示

![image-20200626110413113](HUE.assets/image-20200626110413113.png)

![全搜索](https://cdn.gethue.com/uploads/2015/08/search-full-mode.png)

![分析维度](https://cdn.gethue.com/uploads/2018/08/dashboard_layout_dimensions.gif)

![标记图](https://cdn.gethue.com/uploads/2015/08/search-marker-map.png)

## 应用执行
只实例MapReduce，其他应用类似

![image-20200626110219123](HUE.assets/image-20200626110219123.png)

## 工作流调度

HUE的工作流调度基于Oozie，最新版Oozie支持多流并行执行。

![image-20200626110620856](HUE.assets/image-20200626110620856.png)

![image-20200626111113322](HUE.assets/image-20200626111113322.png)

# 链接

[官网][web]

[官方demo][demo]



[web]:https://gethue.com/
[demo]:https://demo.gethue.com/