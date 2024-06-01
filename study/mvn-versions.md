### 1.简介

versions-maven-plugin插件可以管理项目版本，  
特别是当Maven工程项目中有大量子模块时，  
可以批量修改pom版本号，  
插件会把父模块更新到指定版本号，  
然后更新子模块版本号与父模块相同，  
可以避免手工大量修改和遗漏的问题。

### 2.使用

### 2.1.修改版本号

cmd进入Maven工程根目录，运行命令:

```shell
mvn -f "pom.xml" versions:set -DoldVersion=* -DnewVersion=1.2.0-SNAPSHOT -DprocessAllModules=true -DallowSnapshots=true -DgenerateBackupPoms=true
```

修改成功后，全部模块版本号都变成了1.2.0-SNAPSHAOT。

简化版命令：

```shell
mvn versions:set -DnewVersion=1.2.0-SNAPSHOT
```

该命令和上一条命令等价，  

### 2.2.回退版本号

```shell
mvn versions:revert
```

注意设置generateBackupPoms为true（默认值），  
才会有pom.xml.versionsBackup备份文件，  
否则没有备份文件无法回退版本号。

或者使用版本管理工具提供的撤销功能，  
比如git直接回滚到原始版本：

git reset --hard origin/master

### 2.3.确认修改过的版本号

```shell
mvn versions:commit
```

查看修改后的pom文件，如果没有问题则进行确认，  
该命令会删除修改版本号时生成的pom备份文件。

### 2.4.直接修改版本号，无需确认

设置generateBackupPoms为false，  
则直接修改pom，不会生成备份文件，  
也就不需要使用commit再次确认，  
但是也无法使用revert命令回退版本号。

```shell
mvn versions:set -DnewVersion=1.2.0-SNAPSHOT -DgenerateBackupPoms=false
```

### 3.参数介绍

| 参数                                         | 默认值                   | 说明                           |
| ------------------------------------------ | --------------------- | ---------------------------- |
| allowSnapshots                             | false                 | 是否更新-snapshot快照版             |
| artifactId                                 | ${project.artifactId} | 指定artifactId                 |
| generateBackupPoms                         | true                  | 是否生成备份文件用于回退版本号              |
| groupId                                    | ${project.groupId}    | 指定groupId                    |
| newVersion                                 |                       | 设置的新版本号                      |
| nextSnapshot                               | false                 | 更新版本号为下一个快照版本号               |
| oldVersion                                 | ${project.version}    | 指定需要更新的版本号可以使用缺省'*'          |
| processAllModules                          | false                 | 是否更新目录下所有模块无论是否声明父子节点        |
| processDependencies                        | true                  | 是否更新依赖其的版本号                  |
| processParent                              | true                  | 是否更新父节点的版本号                  |
| processPlugins                             | true                  | 是否更新插件中的版本号                  |
| processProject                             | true                  | 是否更新模块自身的版本号                 |
| removeSnapshot | false                 | 移除snapshot快照版本，使之为release稳定版 |
| updateMatchingVersions                     | true                  | 是否更新在子模块中显式指定的匹配版本(如/项目/版本)  |

### 4.使用技巧

为了更好的使用插件修改版本号，  
减少不必要的版本号修改，  
推荐Maven工程遵循如下规范：  
1.同一项目中所有模块版本保持一致  
2.子模块统一继承父模块的版本  
3.统一在顶层模块Pom的节中定义所有子模块的依赖版本号，子模块中添加依赖时不要添加版本号  
4.开发测试阶段使用SNAPSHOT  
5.生产发布使用RELEASE  
6.新版本迭代只修改父POM中的版本和子模块依赖的父POM版本
