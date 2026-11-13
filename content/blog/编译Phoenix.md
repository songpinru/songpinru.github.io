---
title: "编译Phoenix"
date: 2024-05-29
categories: ["learning-notes"]
tags: ["hadoop", "hbase", "maven", "源码编译"]
---


- 拉取源码

```bash
cd /opt/software/source
git clone https://git-wip-us.apache.org/repos/asf/phoenix.git
```

- 编译

```bash
cd phoenix
mvn clean package -DskipTests -Dhbase.profile=2.2 -Dhbase.version=2.2.4
```
