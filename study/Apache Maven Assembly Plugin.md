# Apache Maven Assembly Plugin

## 支持的打包格式

* zip
* tar
* tar.gz (or tgz)
* tar.bz2 (or tbz2)
* tar.snappy
* tar.xz (or txz)
* jar
* dir
* war
* 自定义（使用xml定义）

## POM格式：

```xml
<project>
	<build>
		<plugins>
			<plugin>
			<!-- 翻译：不需要groupId，他的group是org.apache.maven.plugins，这是默认的-->
				<artifactId>maven-assembly-plugin</artifactId>
				<version>3.3.0</version>
				<configuration>
				<!--入口 -->
					<archive>
						<manifest>
							<mainClass>org.sample.App</mainClass>
						</manifest>
					</archive>
					<!--描述文件 -->
					<descriptorRefs>
						<descriptorRef>jar-with-dependencies</descriptorRef>
					</descriptorRefs>
					<!--descriptor和descriptorRef可以指定多个，并且可以同时存在 -->
					<descriptors>
						<descriptor>src/assembly/src.xml</descriptor>
					</descriptors>
				</configuration>
				<!--执行计划 -->
				<executions>
					<execution>
					<id>make-assembly</id> <!-- this is used for inheritance merges -->
					<phase>package</phase> <!-- 绑定到maven的阶段 -->
					<goals>
						<goal>single</goal>
					</goals>
					</execution>
				</executions>
			</plugin>
		</plugins>
	</build>
</project>
```

descriptorRef:

* [bin](http://maven.apache.org/plugins/maven-assembly-plugin/descriptor-refs.html#bin)

* [jar-with-dependencies](http://maven.apache.org/plugins/maven-assembly-plugin/descriptor-refs.html#jar-with-dependencies)

* [src](http://maven.apache.org/plugins/maven-assembly-plugin/descriptor-refs.html#src)

* [project](http://maven.apache.org/plugins/maven-assembly-plugin/descriptor-refs.html#project)

## goal

| goal                      | description                                                  |
| ------------------------- | ------------------------------------------------------------ |
| assembly:single           | 直接打包，必需的文件在构建开始之前就可用                     |
| assembly:assembly         | Maven在处理程序集之前构建所有包含的POMs，直到打包阶段。      |
| assembly:directory        | 类似于assembly，但是忽略formats，只支持dir                   |
| assembly:directory-single | 直接打包，必需的文件在构建开始之前就可用，且但是忽略formats，只支持dir |
| assembly:help             | 帮助信息                                                     |



```
assembly:single
  Assemble an application bundle or distribution from an assembly descriptor.
  This goal is suitable either for binding to the lifecycle or calling directly
  from the command line (provided all required files are available before the
  build starts, or are produced by another goal specified before this one on the
  command line).
  从程序集描述符中组装应用程序包或分发。
  此目标适合于绑定到生命周期或直接从命令行调用
  （假设所有必需的文件在构建开始之前就可用，或者由在此之前在命令行中指定的另一个目标生成）。
  
assembly:assembly
  Assemble an application bundle or distribution using an assembly descriptor
  from the command line. This goal will force Maven to build all included POMs
  up to the package phase BEFORE the assembly is processed.
  NOTE: This goal should ONLY be run from the command line, and if building a
  multimodule project it should be used from the root POM. Use the
  assembly:single goal for binding your assembly to the lifecycle.
  从命令行使用组装描述符组装应用程序包或发行版。
  这一目标将迫使Maven在处理程序集之前构建所有包含的POMs，直到打包阶段。
  注意:这个目标应该只从命令行运行，如果构建一个多模块项目，则应该从根POM使用它。
  使用程序集:single目标将程序集绑定到生命周期。

assembly:directory
  Like the assembly:attached goal, assemble an application bundle or
  distribution using an assembly descriptor from the command line. This goal
  will force Maven to build all included POMs up to the package phase BEFORE the
  assembly is processed. This goal differs from assembly:assembly in that it
  ignores the <formats/> section of the assembly descriptor, and forces the
  assembly to be created as a directory in the project's build-output directory
  (usually ./target).
  This goal is also functionally equivalent to using the assembly:assembly goal
  in conjunction with the dir assembly format.
  NOTE: This goal should ONLY be run from the command line, and if building a
  multimodule project it should be used from the root POM. Use the
  assembly:directory-single goal for binding your assembly to the lifecycle.
  
  像Assembly：attached目标一样，使用命令行中的程序集描述符来组装应用程序包或分发。
  这个目标将迫使Maven在组装之前在组装阶段之前构建所有包含的POM。
  这个目标与assembly：assembly的不同之处在于，它忽略了程序集描述符的<formats />部分，
  并强制将程序集创建为项目的build-output目录中的目录（通常是./target）。 
  此目标在功能上也等同于结合使用assembly：assembly目标和dir汇编格式。
  注意：此目标仅应从命令行运行，如果要构建多模块项目，则应从根POM使用它。
  使用assembly：directory-single目标将程序集绑定到生命周期。

assembly:directory-single
  Like the assembly:attached goal, assemble an application bundle or
  distribution from an assembly descriptor. This goal is suitable either for
  binding to the lifecycle or calling directly from the command line (provided
  all required files are available before the build starts, or are produced by
  another goal specified before this one on the command line).
  This goal differs from assembly:single in that it ignores the <formats/>
  section of the assembly descriptor, and forces the assembly to be created as a
  directory in the project's build-output directory (usually ./target).
  像Assembly：attached目标一样，从程序集描述符中组装应用程序包或发行版。
  此目标适合于绑定到生命周期或直接从命令行调用（前提是所有必需的文件在构建开始之前就可用，
  或者由在此之前在命令行中指定的另一个目标生成）。 
  这个目标与assembly的不同之处在于：它忽略了程序集描述符的<formats />部分，
  并强制将程序集创建为项目的build-output目录中的目录（通常是./target）。
  
assembly:help
  Display help information on maven-assembly-plugin.
  Call
    mvn assembly:help -Ddetail=true -Dgoal=<goal-name>
  to display parameter details.
  在maven-assembly-plugin上显示帮助信息。
   Call
     mvn assembly：help -Ddetail = true -Dgoal = <goal-name>以显示参数详细信息。

assembly:unpack
  Deprecated. Use org.apache.maven.plugins:maven-dependency-plugin goal: unpack
  or unpack-dependencies instead.
  过期指令，使用org.apache.maven.plugins:maven-dependency-plugin goal: unpack，
  或者unpack-dependencies目标代替

assembly:attached
  Deprecated. Use goal: 'assembly' (from the command line) or 'single' (from a
  lifecycle binding) instead.
  过期指令，使用assembly目标，或者single目标代替
  
assembly:directory-inline
  Deprecated. Use goal: 'directory' (from the command line) or
  'directory-single' (from a lifecycle binding) instead.
  过期指令，使用directory目标，或者directory-single目标代替
```



# 自定义xml

## assembly

程序集定义文件的集合，这些文件通常以存档格式（例如从项目生成的zip，tar或tar.gz）分发。例如，一个项目可以生成一个ZIP程序集，该程序集在根目录中包含一个项目的JAR工件，在lib /目录中包含运行时依赖关系，以及一个用于启动独立应用程序的shell脚本。

| Element                                                      | Type                   | Description                                                  |
| ------------------------------------------------------------ | ---------------------- | ------------------------------------------------------------ |
| `ID`                                                         | `String`               | 设置此程序集的ID。这是该项目中文件的特定程序集的符号名称。另外，除了用于通过将其值附加到生成的归档文件中来明确命名组装好的程序包之外，该ID还在部署时用作工件的分类器。 |
| `formats/format*`                                            | `List<String>`         | **（许多）**指定程序集的格式。通常最好通过目标参数而不是此处指定格式。例如，这允许不同的配置文件生成不同类型的档案。可以提供多种格式，并且Assembly Plugin将为每种所需格式生成一个存档。部署项目时，所有指定的文件格式也将被部署。通过在<format>子元素中提供以下值之一来指定格式：<br />**“ zip”** -创建一个ZIP文件格式<br />**“ tar”** -创建TAR格式<br />**“ tar.gz”**或**“ tgz”** -创建gzip格式的TAR格式<br />**“ tar.bz2”**或**“ tbz2”** -创建bzip格式的TAR格式<br />**“ tar.snappy”** -创建一个活泼的TAR格式<br />**“ tar.xz”**或**“ txz”** -创建xz'd TAR格式<br />**“ jar”** -创建一个JAR格式<br />**“ dir”** -创建爆炸的目录格式<br />**“war”** -创建WAR格式 |
| `includeBaseDirectory`                                       | `boolean`              | 在最终归档文件中包含基本目录。例如，如果要创建一个名为“ your-app”的程序集，则将includeBaseDirectory设置为true将创建一个包含此基本目录的存档。如果将此选项设置为false，则创建的归档文件会将其内容解压缩到当前目录。 **默认值是**：`true`。 |
| `baseDirectory`                                              | `String`               | 设置生成的程序集档案的基本目录。如果未设置，并且includeBaseDirectory == true，则将使用$ {project.build.finalName}。（自2.2-beta-1开始） |
| `includeSiteDirectory`                                       | `boolean`              | 在最终存档中包含站点目录。项目的站点目录位置由Assembly Plugin的configuration中siteDirectory参数确定。 **默认值为**：`false`。 |
| `containerDescriptorHandlers / containerDescriptorHandler *` | `List`                 | **（许多）**从常规归档流中过滤掉各种容器描述符的组件集，因此可以对其进行汇总然后添加。 |
| `moduleSets / moduleSet *`                                   | `List <<ModuleSet>`    | **（许多）**指定程序集中要包含的模块文件。通过提供一个或多个<moduleSet>子元素来指定moduleSet。 |
| `fileSets / fileSet *`                                       | `List<FileSet>`        | **（许多）**指定程序集中要包括的文件组。通过提供一个或多个<fileSet>子元素来指定fileSet。 |
| `files/ file*`                                               | `List<FileItem>`       | **（许多）**指定程序集中要包含的单个文件。通过提供一个或多个<file>子元素来指定文件。 |
| `dependencySets / dependencySet *`                           | `List <DependencySet>` | **（许多）**指定要在程序集中包括的依赖项。通过提供一个或多个<dependencySet>子元素来指定dependencySet。 |
| `repositories / repository* `                                | `List<Repository>`     | **（许多）**指定程序集中要包含的存储库文件。通过提供一个或多个<repository>子元素来指定存储库。 |
| `componentDescriptors / componentDescriptor *`               | `List<String>`         | **（许多）**指定要包含在程序集中的共享组件xml文件的位置。指定的位置必须相对于描述符的基本位置。如果描述符是通过类路径中的<descriptorRef />元素找到的，则它指定的任何组件也都可以在类路径中找到。如果通过<descriptor />元素通过路径名找到它，则此处的值将解释为相对于基于项目的路径。当找到多个componentDescriptor时，它们的内容将合并。查看 [描述符组件](http://maven.apache.org/plugins/maven-assembly-plugin/assembly-component.html)以获取更多信息。通过提供一个或多个<componentDescriptor>子元素来指定componentDescriptor。 |



### containerDescriptorHandler

为进入程序集归档文件的文件配置过滤器，以启用各种类型的描述符片段的聚合，例如components.xml，web.xml等。

| Element         | Type     | Description                                  |
| --------------- | -------- | -------------------------------------------- |
| `handlerName`   | `String` | 处理程序的plexus角色提示，用于从容器中查找。 |
| `configuration` | `DOM`    | 处理程序的配置选项。                         |

[containerDescriptorHandler详解](#containerDescriptorHandler详解)



###  moduleSet

moduleSet表示项目的pom.xml中存在的一个或多个项目<module>。这使您可以包括属于项目<modules>的源或二进制文件。

| Element                 | Type             | Description                                                  |
| ----------------------- | ---------------- | ------------------------------------------------------------ |
| `useAllReactorProjects` | `boolean`        | 如果设置为true，则该插件将包括当前反应堆中的所有项目，以便在此ModuleSet中进行处理。这些将受包含/排除规则的约束。（自2.2开始）<br/>**默认值为**：`false` |
| `includeSubModules`     | `boolean`        | 如果设置为false，则插件将在此ModuleSet中排除子模块。否则，它将处理所有子模块，每个子模块都要包含/排除规则。（自2.2-beta-1开始）<br/>**默认值为**：`true`。 |
| `includes/include*`     | `List<String>`   | **（许多）**存在<include>子元素时，它们定义要包含的一组项目坐标。如果不存在，则<includes>代表所有有效值。工件坐标可以以简单的groupId：artifactId形式给出，或者可以以groupId：artifactId：type [：classifier]：version形式完全限定。此外，可以使用通配符，如*：maven- * |
| `excludes/exclude*`     | `List<String>`   | **（许多）**存在<exclude>子元素时，它们定义一组要排除的项目工件坐标。如果不存在，则<excludes>表示没有排除。工件坐标可以以简单的groupId：artifactId形式给出，或者可以以groupId：artifactId：type [：classifier]：version形式完全限定。此外，可以使用通配符，如*：maven- * |
| `sources`               | `ModuleSources`  | 如果存在此选项，则插件将在结果程序集中包含该集合中所包含模块的源文件 |
| `binaries`              | `ModuleBinaries` | 如果存在此选项，则插件将在结果程序集中包含该集合中所包含模块的二进制文件。 |



#### sources

| Element                       | Type            | Description                                                  |
| ----------------------------- | --------------- | ------------------------------------------------------------ |
| `useDefaultExcludes`          | `boolean`       | 计算受此集合影响的文件时，是否应使用标准排除模式，例如与CVS和Subversion元数据文件匹配的那些。为了向后兼容，默认值为true。（自2.2-beta-1开始） **默认值为**：`true`。 |
| `outputDirectory`             | `String`        | 设置相对于程序集根目录根目录的输出目录。例如，“ log”会将指定的文件放在log目录中。 |
| `includes/include*`           | `List<String>`  | **（许多）**存在<include>子元素时，它们定义一组要包括的文件和目录。如果不存在，则<includes>代表所有有效值。 |
| `excludes/exclude*`           | `List<String>`  | **（许多 ）**存在<exclude>子元素时，它们定义一组要排除的文件和目录。如果不存在，则<excludes>表示没有排除。 |
| `fileMode`                    | `String`        | 类似于UNIX权限，设置包含文件的文件模式。这是一个重要的价值。格式：（User）（Group）（Other），其中每个组件的总和为Read = 4，Write = 2和Execute =1。例如，值0644转换为User read-write，Group和Other只读。默认值为0644 [（有关Unix样式的更多权限）](http://www.onlamp.com/pub/a/bsd/2000/09/06/FreeBSD_Basics.html) |
| `directoryMode`               | `String`        | 类似于UNIX权限，设置包含目录的目录模式。这是一个重要的价值。格式：（User）（Group）（Other），其中每个组件的总和为Read = 4，Write = 2和Execute =1。例如，值0755转换为User read-write，Group和Other只读。默认值为0755。 [（有关Unix风格的权限的更多信息）](http://www.onlamp.com/pub/a/bsd/2000/09/06/FreeBSD_Basics.html) |
| `fileSets / fileSet *`        | `List<FileSet>` | **（许多）**指定每个包括的模块中的文件组要包括在程序集中。通过提供一个或多个<fileSet>子元素来指定fileSet。（自2.2-beta-1开始） |
| `includeModuleDirectory`      | `boolean`       | 指定是否将模块的finalName附加到应用于该模块的任何fileSet的outputDirectory值之前。（自2.2-beta-1开始） **默认值为**：`true`。 |
| `excludeSubModuleDirectories` | `boolean`       | 指定是否应将当前模块下的子模块目录从应用于该模块的文件集中排除。如果仅打算复制与此ModuleSet匹配的确切模块列表的源，而忽略（或单独处理）当前目录下目录中存在的模块，则这可能很有用。（自2.2-beta-1开始） **默认值为**：`true`。 |
| `outputDirectoryMapping`      | `boolean`       | 设置此程序集中包含的所有模块基本目录的映射模式。注意：仅当includeModuleDirectory == true时才使用此字段。默认值为2.2-beta-1中模块的$ {artifactId}，以及后续版本中的$ {module.artifactId}。（自2.2-beta-1开始） **默认值为**：`$ {module.artifactId}`。 |

#### binaries

| Element                          | Type                  | Description                                                  |
| -------------------------------- | --------------------- | ------------------------------------------------------------ |
| `outputDirectory`                | `String`              | 设置相对于程序集根目录根目录的输出目录。例如，“ log”会将指定的文件放在日志目录中，该目录直接位于存档根目录下。 |
| `includes/include*`              | `List<String>`        | **（许多）**当存在<include>子元素时，它们定义了一组要包含的伪像坐标。如果不存在，则<includes>代表所有有效值。工件坐标可以以简单的groupId：artifactId形式给出，或者可以以groupId：artifactId：type [：classifier]：version形式完全限定。此外，可以使用通配符，如*：maven- * |
| `excludes/exclude*`              | `List<String>`        | **（许多）**存在<exclude>子元素时，它们定义一组要排除的依赖项工件坐标。如果不存在，则<excludes>表示没有排除。工件坐标可以以简单的groupId：artifactId形式给出，或者可以以groupId：artifactId：type [：classifier]：version形式完全限定。此外，可以使用通配符，如*：maven- * |
| `fileMode`                       | `String`              | 类似于UNIX权限，设置包含文件的文件模式。这是一个重要的价值。格式：（User）（Group）（Other），其中每个组件的总和为Read = 4，Write = 2和Execute =1。例如，值0644转换为User read-write，Group和Other只读。默认值为0644 [（有关Unix样式的更多权限）](http://www.onlamp.com/pub/a/bsd/2000/09/06/FreeBSD_Basics.html) |
| `directoryMode`                  | `String`              | 类似于UNIX权限，设置包含目录的目录模式。这是一个重要的价值。格式：（User）（Group）（Other），其中每个组件的总和为Read = 4，Write = 2和Execute =1。例如，值0755转换为User read-write，Group和Other只读。默认值为0755。 [（有关Unix风格的权限的更多信息）](http://www.onlamp.com/pub/a/bsd/2000/09/06/FreeBSD_Basics.html) |
| `attachmentClassifier`           | `String`              | 指定后，attachmentClassifier将使汇编器查看附加到模块的工件，而不是主项目工件。如果可以找到与指定分类器匹配的附加工件，则将使用它；否则，它将使用它。否则，将引发异常。（自2.2-beta-1开始） |
| `includeDependencies`            | `boolean`             | 如果设置为true，则该插件将包括此处包含的项目模块的直接和传递依赖关系。否则，它将仅包括模块软件包。 **默认值是**：`真`。 |
| `dependencySets/ dependencySet*` | `List<DependencySet>` | **（许多）**指定要在程序集中包含模块的哪些依赖项。通过提供一个或多个<dependencySet>子元素来指定dependencySet。（自2.2-beta-1开始） |
| `unpack`                         | `boolean`             | 如果设置为true，则此属性会将所有模块包解压缩到指定的输出目录中。设置为false时，模块软件包将作为归档文件（jar）包含在内。 **默认值是**：`真`。 |
| `unpackOptions`                  | `UnpackOptions`       | 允许为从模块工件中解包的项目指定包含和排除以及过滤选项。（自2.2-beta-1开始） |
| `outputFileNameMapping`          | `String`              | 设置此程序集中包含的所有NON-UNPACKED依赖项的映射模式。（因为2.2-beta-2；从2.2-beta-1开始，使用$ {artifactId}-$ {version} $ {dashClassifier？}。$ {extension}作为默认值）注：如果dependencySet指定unpack == true，则outputFileNameMapping将不被使用；在这种情况下，请使用outputDirectory。有关outputFileNameMapping参数中可用条目的更多详细信息，请参见插件FAQ。 **默认值为**：`$ {module.artifactId}-$ {module.version} $ {dashClassifier？}。$ {module.extension}`。 |

### fileSet

| Element                                                | Type           | Description                                                  |
| ------------------------------------------------------ | -------------- | ------------------------------------------------------------ |
| `useDefaultExcludes`                                   | `boolean`      | 计算受此集合影响的文件时，是否应使用标准排除模式，例如与CVS和Subversion元数据文件匹配的那些。为了向后兼容，默认值为true。（自2.2-beta-1开始） **默认值为**：`true`。 |
| `outputDirectory`                                      | `String`       | 设置相对于程序集根目录根目录的输出目录。例如，“ log”会将指定的文件放在log目录中。 |
| `includes/include*`                                    | `List<String>` | **（许多）**存在<include>子元素时，它们定义一组要包括的文件和目录。如果不存在，则<includes>代表所有有效值。 |
| `excludes/exclude*`                                    | `List<String>` | **（许多）**存在<exclude>子元素时，它们定义一组要排除的文件和目录。如果不存在，则<excludes>表示没有排除。 |
| `fileMode`                                             | `String`       | 类似于UNIX权限，设置包含文件的文件模式。这是一个重要的价值。格式：（User）（Group）（Other），其中每个组件的总和为Read = 4，Write = 2和Execute =1。例如，值0644转换为User read-write，Group和Other只读。默认值为0644。 [（有关Unix样式的更多权限）](http://www.onlamp.com/pub/a/bsd/2000/09/06/FreeBSD_Basics.html) |
| `directoryMode`                                        | `String`       | 类似于UNIX权限，设置包含目录的目录模式。这是一个重要的价值。格式：（User）（Group）（Other），其中每个组件的总和为Read = 4，Write = 2和Execute =1。例如，值0755转换为User read-write，Group和Other只读。默认值为0755。 [（有关Unix风格的权限的更多信息）](http://www.onlamp.com/pub/a/bsd/2000/09/06/FreeBSD_Basics.html) |
| `directory`                                            | `String`       | 从模块目录设置绝对或相对位置。例如，“ src / main / bin”将选择在其中定义此依赖项的项目的此子目录。 |
| `lineEnding`                                           | `String`       | 设置此fileSet中文件的行尾。有效值：<br />**“keep”** -保留所有行尾<br />**“ unix”** -使用Unix样式的行尾（即“ \ n”）<br />**“ lf”** -使用单个换行符（例如“ \ n”）<br />**“ dos”** -使用DOS- / Windows样式的行尾（例如“ \ r \ n”）<br />**“ windows”** -使用DOS- / Windows样式的行尾（例如“ \ r \ n”）<br />**“ crlf”** -使用回车符，换行符（例如“ \ r \ n”） |
| `filtered`                                             | `boolean`      | 是否使用bulid中的属性在复制文件时过滤符号。（自2.2-beta-1开始） **默认值为**：`false`。 |
| `nonFilteredFileExtensions/ nonFilteredFileExtension*` | `List<String>` | **（许多）**其他文件扩展名不应用过滤（自3.2.0版开始）        |

### dependencySet

| Element                     | Type            | Description                                                  |
| --------------------------- | --------------- | ------------------------------------------------------------ |
| `outputDirectory`           | `String`        | 设置相对于程序集根目录根目录的输出目录。例如，“ log”会将指定的文件放在日志目录中，该目录直接位于存档根目录下。 |
| `includes/include*`         | `List<String>`  | **（许多）**当存在<include>子元素时，它们定义了一组要包含的伪像坐标。如果不存在，则<includes>代表所有有效值。工件坐标可以以简单的groupId：artifactId形式给出，或者可以以groupId：artifactId：type [：classifier]：version形式完全限定。此外，可以使用通配符，如*：maven- * |
| `excludes/exclude*`         | `List<String>`  | **（许多）**存在<exclude>子元素时，它们定义一组要排除的依赖项工件坐标。如果不存在，则<excludes>表示没有排除。工件坐标可以以简单的groupId：artifactId形式给出，或者可以以groupId：artifactId：type [：classifier]：version形式完全限定。此外，可以使用通配符，如*：maven- * |
| `fileMode`                  | `String`        | 类似于UNIX权限，设置包含文件的文件模式。这是一个重要的价值。格式：（User）（Group）（Other），其中每个组件的总和为Read = 4，Write = 2和Execute =1。例如，值0644转换为User read-write，Group和Other只读。默认值为0644 [（有关Unix样式的更多权限）](http://www.onlamp.com/pub/a/bsd/2000/09/06/FreeBSD_Basics.html) |
| `directoryMode`             | `String`        | 类似于UNIX权限，设置包含目录的目录模式。这是一个重要的价值。格式：（User）（Group）（Other），其中每个组件的总和为Read = 4，Write = 2和Execute =1。例如，值0755转换为User read-write，Group和Other只读。默认值为0755。 [（有关Unix风格的权限的更多信息）](http://www.onlamp.com/pub/a/bsd/2000/09/06/FreeBSD_Basics.html) |
| `useStrictFiltering`        | `boolean`       | 当指定为true时，任何在装配创建期间未用于过滤实际工件的包含/排除模式都将导致构建失败并显示错误。这是为了突出显示过时的包含或排除，或者表示装配描述符配置不正确。（自2.2开始） **默认值为**：`false`。 |
| `outputFileNameMapping`     | `String`        | 设置此程序集中包含的所有依赖项的映射模式。（自2.2-beta-2开始； 2.2-beta-1使用$ {artifactId}-$ {version} $ {dashClassifier？}。$ {extension}作为默认值）。有关outputFileNameMapping参数中可用条目的更多详细信息，请参见插件FAQ。 **默认值为**：`${artifact.artifactId}-${artifact.version}${dashClassifier？}.$ {artifact.extension}`。 |
| `unpack`                    | `boolean`       | 如果设置为true，则此属性会将所有依赖项解压缩到指定的输出目录中。设置为false时，依赖项将作为存档（jar）包含在内。只能解压缩jar，zip，tar.gz和tar.bz档案。 **默认值为**：`false`。 |
| `unpackOptions`             | `UnpackOptions` | 允许为从依赖项工件中解包的项目指定包含和排除以及过滤选项。（自2.2-beta-1开始） |
| `scope`                     | `String`        | 设置此DependencySet的依赖项范围。 **默认值为**：`runtime`。  |
| `useProjectArtifact`        | `boolean`       | 确定当前项目构建期间生成的工件是否应包括在此依赖项集中。（自2.2-beta-1开始） **默认值为**：`true`。 |
| `useProjectAttachments`     | `boolean`       | 确定当前项目构建期间生成的附加工件是否应包括在此依赖项集中。（自2.2-beta-1开始） **默认值为**：`false`。 |
| `useTransitiveDependencies` | `boolean`       | 确定是否将传递依赖项包含在当前依赖项集的处理中。如果为true，则除了主要项目依赖项外，includes / excludes / useTransitiveFiltering还将应用于传递依赖项。如果为false，则useTransitiveFiltering毫无意义，并且包含/排除项仅影响项目的直接依赖关系。默认情况下，此值为true。（自2.2-beta-1开始） **默认值为**：`true`。 |
| `useTransitiveFiltering`    | `boolean`       | 确定此依赖集中的包含/排除模式是否将应用于给定工件的传递路径。如果为true，并且当前工件是由另一个与包含或排除模式匹配的工件带来的传递性依赖关系，则当前工件也具有相同的包含/排除逻辑。默认情况下，此值是false，以便保留与2.1版的向后兼容性。这意味着包含/排除项仅直接应用于当前工件，而不应用于引入工件的可传递工件集。（自2.2-beta-1起） **默认值为**：`false`。 |

#### unpackOptions

指定用于包括/排除/过滤从存档中提取的项目的选项。（自2.2-beta-1开始）

| Element                                                | Type           | Description                                                  |
| ------------------------------------------------------ | -------------- | ------------------------------------------------------------ |
| `includes/include*`                                    | `List<String>` | **（许多）**文件和/或目录模式集，用于在解压缩档案时包含在档案中的匹配项目。每个项目都指定为<include> some / path </ include>（从2.2-beta-1开始） |
| `excludes/exclude*`                                    | `List<String>` | **（许多）**文件和/或目录模式集，用于在解压缩存档时将其排除在存档之外的匹配项。每个项目均指定为<exclude> some / path </ exclude>（自2.2-beta-1开始） |
| `filtered`                                             | `boolean`      | 是否使用构建配置中的属性过滤从归档文件中解压缩的文件中的符号。（自2.2-beta-1开始） **默认值为**：`false`。 |
| `nonFilteredFileExtensions/ nonFilteredFileExtension*` | `List<String>` | **（许多）**其他文件扩展名不应用过滤（自3.2.0版开始）        |
| `lineEnding`                                           | `String`       | 设置文件的行尾。（自2.2开始）有效值：**“保持”** -保留所有行尾**“ unix”** -使用Unix样式的行尾**“ lf”** -使用单个换行符**“ dos”** -使用DOS样式的行尾**“** crlf **”** -使用回车符，换行符 |
| `useDefaultExcludes`                                   | `boolean`      | 计算受此集合影响的文件时，是否应使用标准排除模式，例如与CVS和Subversion元数据文件匹配的那些。为了向后兼容，默认值为true。（自2.2开始） **默认值为**：`true`。 |
| `encoding`                                             | `String`       | 允许为支持指定编码的取消归档者指定解压缩归档文件时使用的编码。如果未指定，将使用存档器默认值。存档程序默认值通常表示合理（现代）的值。 |

### file

文件允许单个文件包含，并带有选项来更改fileSet不支持的目标文件名。注意：需要一个或多个来源

| Element           | Type           | Description                                                  |
| ----------------- | -------------- | ------------------------------------------------------------ |
| `source`          | `String`       | 从文件的模块目录设置要包含在程序集中的绝对或相对路径。       |
| `sources/source*` | `List<String>` | **（许多）**来自文件模块目录的一组绝对或相对路径被组合并包含在装配中。 |
| `outputDirectory` | `String`       | 设置相对于程序集根目录根目录的输出目录。例如，“ log”会将指定的文件放在log目录中。 |
| `destName`        | `String`       | 在outputDirectory中设置目标文件名。默认名称与源文件的名称相同。 |
| `fileMode`        | `String`       | 类似于UNIX权限，设置包含文件的文件模式。这是一个重要的价值。格式：（User）（Group）（Other），其中每个组件的总和为Read = 4，Write = 2和Execute =1。例如，值0644转换为User read-write，Group和Other只读。默认值为0644 [（有关Unix样式的更多权限）](http://www.onlamp.com/pub/a/bsd/2000/09/06/FreeBSD_Basics.html) |
| `lineEnding`      | `String`       | 设置此文件中文件的行尾。有效值为：<br />**“keep”** -保留所有行尾<br />**“ unix”** -使用Unix样式的行尾（即“ \ n”）<br />**“ lf”** -使用单个换行符（例如“ \ n”）<br />**“ dos”** -使用DOS- / Windows样式的行尾（例如“ \ r \ n”）<br />**“ windows”** -使用DOS- / Windows样式的行尾（例如“ \ r \ n”）<br />**“ crlf”** -使用回车符，换行符（例如“ \ r \ n”） |
| `filtered`        | `boolean`      | 设置是否确定文件是否被过滤。 **默认值为**：`false`。         |

### repository

定义要包含在程序集中的Maven存储库。可以包含在存储库中的工件是项目的依赖工件。创建的存储库包含所需的元数据条目，还包含sha1和md5校验和。这对于创建将部署到内部存储库的归档文件很有用。

**注意：**当前，仅允许来自中央存储库的工件。

| Element                                                | Type                          | Description                                                  |
| ------------------------------------------------------ | ----------------------------- | ------------------------------------------------------------ |
| `outputDirectory`                                      | `String`                      | 设置相对于程序集根目录根目录的输出目录。例如，“ log”会将指定的文件放在日志目录中，该目录直接位于存档根目录下。 |
| `includes/include*`                                    | `List<String>`                | **（许多）**当存在<include>子元素时，它们定义了一组要包含的伪像坐标。如果不存在，则<includes>代表所有有效值。工件坐标可以以简单的groupId：artifactId形式给出，或者可以以groupId：artifactId：type [：classifier]：version形式完全限定。此外，可以使用通配符，如*：maven- * |
| `excludes/exclude*`                                    | `List<String>`                | **（许多）**存在<exclude>子元素时，它们定义一组要排除的依赖项工件坐标。如果不存在，则<excludes>表示没有排除。工件坐标可以以简单的groupId：artifactId形式给出，或者可以以groupId：artifactId：type [：classifier]：version形式完全限定。此外，可以使用通配符，如*：maven- * |
| `fileMode`                                             | `String`                      | 类似于UNIX权限，设置包含文件的文件模式。这是一个重要的价值。格式：（User）（Group）（Other），其中每个组件的总和为Read = 4，Write = 2和Execute =1。例如，值0644转换为User read-write，Group和Other只读。默认值为0644 [（有关Unix样式的更多权限）](http://www.onlamp.com/pub/a/bsd/2000/09/06/FreeBSD_Basics.html) |
| `directoryMode`                                        | `String`                      | 类似于UNIX权限，设置包含目录的目录模式。这是一个重要的价值。格式：（User）（Group）（Other），其中每个组件的总和为Read = 4，Write = 2和Execute =1。例如，值0755转换为User read-write，Group和Other只读。默认值为0755。 [（有关Unix风格的权限的更多信息）](http://www.onlamp.com/pub/a/bsd/2000/09/06/FreeBSD_Basics.html) |
| `includeMetadata`                                      | `boolean`                     | 如果设置为true，则此属性将触发创建存储库元数据，这将使该存储库可用作功能性远程存储库。 **默认值为**：`false`。 |
| `groupVersionAlignments/ groupVersionAlignment*` | `List<GroupVersionAlignment>` | **（许多）**指定您要将一组工件与指定版本对齐。通过提供一个或多个<groupVersionAlignment>子元素来指定groupVersionAlignment。 |
| `scope`                                                | `String`                      | 指定此存储库中包含的工件的范围。（自2.2-beta-1开始） **默认值为**：`runtime`。 |

#### groupVersionAlignment

允许一组工件与指定版本对齐。

| Element             | Type           | Description                                                  |
| ------------------- | -------------- | ------------------------------------------------------------ |
| `id`                | `String`       | 要为其对齐版本的工件的groupId。                              |
| `version`           | `String`       | 您要与此组对齐的版本。                                       |
| `excludes/exclude*` | `List<String>` | **（许多）**当存在<exclude>子元素时，它们定义要排除的工件的artifactIds。如果不存在，则<excludes>表示没有排除。通过提供一个或多个<exclude>子元素来指定排除。 |

## 模板

```xml
<assembly xmlns="http://maven.apache.org/ASSEMBLY/2.1.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/ASSEMBLY/2.1.0 http://maven.apache.org/xsd/assembly-2.1.0.xsd">
  <id/>
  <formats/>
  <includeBaseDirectory/>
  <baseDirectory/>
  <includeSiteDirectory/>
  <containerDescriptorHandlers>
    <containerDescriptorHandler>
      <handlerName/>
      <configuration/>
    </containerDescriptorHandler>
  </containerDescriptorHandlers>
  <moduleSets>
    <moduleSet>
      <useAllReactorProjects/>
      <includeSubModules/>
      <includes/>
      <excludes/>
      <sources>
        <useDefaultExcludes/>
        <outputDirectory/>
        <includes/>
        <excludes/>
        <fileMode/>
        <directoryMode/>
        <fileSets>
          <fileSet>
            <useDefaultExcludes/>
            <outputDirectory/>
            <includes/>
            <excludes/>
            <fileMode/>
            <directoryMode/>
            <directory/>
            <lineEnding/>
            <filtered/>
            <nonFilteredFileExtensions/>
          </fileSet>
        </fileSets>
        <includeModuleDirectory/>
        <excludeSubModuleDirectories/>
        <outputDirectoryMapping/>
      </sources>
      <binaries>
        <outputDirectory/>
        <includes/>
        <excludes/>
        <fileMode/>
        <directoryMode/>
        <attachmentClassifier/>
        <includeDependencies/>
        <dependencySets>
          <dependencySet>
            <outputDirectory/>
            <includes/>
            <excludes/>
            <fileMode/>
            <directoryMode/>
            <useStrictFiltering/>
            <outputFileNameMapping/>
            <unpack/>
            <unpackOptions>
              <includes/>
              <excludes/>
              <filtered/>
              <nonFilteredFileExtensions/>
              <lineEnding/>
              <useDefaultExcludes/>
              <encoding/>
            </unpackOptions>
            <scope/>
            <useProjectArtifact/>
            <useProjectAttachments/>
            <useTransitiveDependencies/>
            <useTransitiveFiltering/>
          </dependencySet>
        </dependencySets>
        <unpack/>
        <unpackOptions>
          <includes/>
          <excludes/>
          <filtered/>
          <nonFilteredFileExtensions/>
          <lineEnding/>
          <useDefaultExcludes/>
          <encoding/>
        </unpackOptions>
        <outputFileNameMapping/>
      </binaries>
    </moduleSet>
  </moduleSets>
  <fileSets>
    <fileSet>
      <useDefaultExcludes/>
      <outputDirectory/>
      <includes/>
      <excludes/>
      <fileMode/>
      <directoryMode/>
      <directory/>
      <lineEnding/>
      <filtered/>
      <nonFilteredFileExtensions/>
    </fileSet>
  </fileSets>
  <files>
    <file>
      <source/>
      <sources/>
      <outputDirectory/>
      <destName/>
      <fileMode/>
      <lineEnding/>
      <filtered/>
    </file>
  </files>
  <dependencySets>
    <dependencySet>
      <outputDirectory/>
      <includes/>
      <excludes/>
      <fileMode/>
      <directoryMode/>
      <useStrictFiltering/>
      <outputFileNameMapping/>
      <unpack/>
      <unpackOptions>
        <includes/>
        <excludes/>
        <filtered/>
        <nonFilteredFileExtensions/>
        <lineEnding/>
        <useDefaultExcludes/>
        <encoding/>
      </unpackOptions>
      <scope/>
      <useProjectArtifact/>
      <useProjectAttachments/>
      <useTransitiveDependencies/>
      <useTransitiveFiltering/>
    </dependencySet>
  </dependencySets>
  <repositories>
    <repository>
      <outputDirectory/>
      <includes/>
      <excludes/>
      <fileMode/>
      <directoryMode/>
      <includeMetadata/>
      <groupVersionAlignments>
        <groupVersionAlignment>
          <id/>
          <version/>
          <excludes/>
        </groupVersionAlignment>
      </groupVersionAlignments>
      <scope/>
    </repository>
  </repositories>
  <componentDescriptors/>
</assembly>
```

# xml模板

```xml
<assembly xmlns="http://maven.apache.org/ASSEMBLY/2.1.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/ASSEMBLY/2.1.0 http://maven.apache.org/xsd/assembly-2.1.0.xsd">
    <!--
        设置此程序集的标识。这是来自此项目的特定文件组合的符号名称。此外，除了用于通过将生成的归档的值附加到组合包以明确命名组合包之外，该ID在部署时用作工件的分类器。
    -->
    <!--string-->
    <id/>
    <!--
        (许多） 指定程序集的格式。通过目标参数而不是在这里指定格式通常会更好。例如，允许不同的配置文件生成不同类型的档案。
        可以提供多种格式，装配体插件将生成每种所需格式的档案。部署项目时，所有指定的文件格式也将被部署。
        通过在<format>子元素中提供以下值之一来指定格式：
        “zip” - 创建一个ZIP文件格式
        “tar” - 创建一个TAR格式
        “tar.gz”或“tgz” - 创建一个gzip'd TAR格式
        “tar.bz2”或“tbz2” - 创建一个bzip'd TAR格式
        “tar.snappy” - 创建一个灵活的TAR格式
        “tar.xz”或“txz” - 创建一个xz'd TAR格式
        “jar” - 创建一个JAR格式
        “dir” - 创建分解的目录格式
        “war” - 创建一个WAR格式
    -->
    <!--List<String>-->
    <formats/>
    <!--
        在最终归档中包含一个基本目录。例如，如果您正在创建一个名为“your-app”的程序集，则将includeBaseDirectory设置为true将创建一个包含此基本目录的归档文件。
        如果此选项设置为false，则创建的存档将其内容解压缩到当前目录。
        默认值是：true。
    -->
    <!--boolean-->
    <includeBaseDirectory/>
    <!--
        设置生成的程序集归档的基本目录。如果没有设置，并且includeBaseDirectory == true，则将使用$ {project.build.finalName}。（从2.2-beta-1开始）
    -->
    <!--string-->
    <baseDirectory/>
    <!--
        在最终档案中包含一个网站目录。项目的站点目录位置由Assembly Plugin的siteDirectory参数确定。
        默认值是：false。
    -->
    <!--boolean-->
    <includeSiteDirectory/>

    <!--
        （许多） 从常规归档流中过滤各种容器描述符的组件集合，因此可以将它们聚合然后添加。
    -->
    <!--List<ContainerDescriptorHandlerConfig>-->
    <containerDescriptorHandlers>
        <!--
            配置文件头部的过滤器，以启用各种类型的描述符片段（如components.xml，web.xml等）的聚合。
        -->
        <containerDescriptorHandler>
            <!--
                处理程序的plexus角色提示，用于从容器中查找。
            -->
            <!--string-->
            <handlerName/>
            <!--
                处理程序的配置选项。
            -->
            <!--DOM-->
            <configuration/>
        </containerDescriptorHandler>
    </containerDescriptorHandlers>
    <!--
        （许多） 指定在程序集中包含哪些模块文件。moduleSet是通过提供一个或多个<moduleSet>子元素来指定的。
    -->
    <!--List<ModuleSet>-->
    <moduleSets>
        <!--
            moduleSet表示一个或多个在项目的pom.xml中存在的<module>项目。这使您可以包含属于项目<modules>的源代码或二进制文件。
            注意：从命令行使用<moduleSets>时，需要先通过“mvn package assembly：assembly”来传递包阶段。这个bug计划由Maven 2.1解决。
        -->
        <moduleSet>
            <!--
                如果设置为true，则该插件将包含当前反应堆中的所有项目，以便在此ModuleSet中进行处理。这些将被 纳入/排除(includes/excludes) 规则。（从2.2开始）
                默认值是：false。
            -->
            <!--boolean-->
            <useAllReactorProjects/>
            <!--
                如果设置为false，则该插件将从该ModuleSet中排除子模块的处理。否则，它将处理所有子模块，每个子模块都要遵守包含/排除规则。（从2.2-beta-1开始）
                默认值是：true。
            -->
            <!--boolean-->
            <includeSubModules/>
            <!--
                （许多） 当存在<include>子元素时，它们定义一组包含的项目坐标。如果不存在，则<includes>表示所有有效值。
                工件坐标可以以简单的groupId：artifactId形式给出，或者可以以groupId：artifactId：type [：classifier]：version的形式完全限定。
                另外，可以使用通配符，如*：maven- *
            -->
            <!--List<String>-->
            <includes/>
            <!--
                （许多） 当存在<exclude>子元素时，它们定义一组要排除的项目工件坐标。如果不存在，则<excludes>不表示排除。
                工件坐标可以以简单的groupId：artifactId形式给出，或者可以以groupId：artifactId：type [：classifier]：version的形式完全限定。
                另外，可以使用通配符，如*：maven- *
            -->
            <!--List<String>-->
            <excludes/>
            <!--
                当存在这个时，插件将在生成的程序集中包含这个集合中包含的模块的源文件。
                包含用于在程序集中包含项目模块的源文件的配置选项。
            -->
            <!--ModuleSources-->
            <sources>
                <!--
                    在计算受该集合影响的文件时，是否应该使用标准排除模式，例如那些匹配CVS和Subversion元数据文件的排除模式。为了向后兼容，默认值是true。（从2.2-beta-1开始）
                    默认值是：true。
                -->
                <!--boolean-->
                <useDefaultExcludes/>
                <!--
                    设置输出目录相对于程序集根目录的根目录。例如，“日志”将把指定的文件放在日志目录中。
                -->
                <!--string-->
                <outputDirectory/>
                <!--
                    （许多） 当<include>子元素存在时，它们定义一组要包含的文件和目录。如果不存在，则<includes>表示所有有效值。
                -->
                <!--List<String>-->
                <includes/>
                <!--
                    （许多） 当存在<exclude>子元素时，它们定义一组要排除的文件和目录。如果不存在，则<excludes>不表示排除。
                -->
                <!--List<String>-->
                <excludes/>
                <!--
                    与UNIX权限类似，设置所包含文件的文件模式。这是一个 OCTAL VALUE。格式：（用户）（组）（其他）其中每个组件是Read = 4，Write = 2和Execute = 1的总和。
                    例如，值0644转换为用户读写，组和其他只读。默认值是0644
                -->
                <!--string-->
                <fileMode/>
                <!--
                    与UNIX权限类似，设置包含的目录的目录模式。这是一个 OCTAL VALUE。格式：（用户）（组）（其他）[Format: (User)(Group)(Other) ] 其中每个组件是Read = 4，Write = 2和Execute = 1的总和。
                    例如，值0755转换为用户读写，Group和其他只读。默认值是0755.
                -->
                <!--string-->
                <directoryMode/>
                <!--
                    （许多） 指定包含在程序集中的每个包含模块的哪些文件组。fileSet通过提供一个或多个<fileSet>子元素来指定。（从2.2-beta-1开始）
                -->
                <!--List<FileSet>-->
                <fileSets>
                    <!--
                        fileSet允许将文件组包含到程序集中。
                    -->
                    <fileSet>
                        <!--
                            在计算受该集合影响的文件时，是否应该使用标准排除模式，例如那些匹配CVS和Subversion元数据文件的排除模式。为了向后兼容，默认值是true。（从2.2-beta-1开始）
                            默认值是：true。
                        -->
                        <!--boolean-->
                        <useDefaultExcludes/>
                        <!--
                            设置输出目录相对于程序集根目录的根目录。例如，“日志”将把指定的文件放在日志目录中。
                        -->
                        <!--string-->
                        <outputDirectory/>
                        <!--
                            （许多） 当<include>子元素存在时，它们定义一组要包含的文件和目录。如果不存在，则<includes>表示所有有效值。
                        -->
                        <!--List<String>-->
                        <includes/>
                        <!--
                            （许多） 当存在<exclude>子元素时，它们定义一组要排除的文件和目录。如果不存在，则<excludes>不表示排除。
                        -->
                        <!--List<String>-->
                        <excludes/>
                        <!--
                            与UNIX权限类似，设置所包含文件的文件模式。这是一个 OCTAL VALUE。格式：（用户）（组）（其他）其中每个组件是Read = 4，Write = 2和Execute = 1的总和。
                            例如，值0644转换为用户读写，组和其他只读。默认值是0644.
                        -->
                        <!--string-->
                        <fileMode/>
                        <!--
                            与UNIX权限类似，设置包含的目录的目录模式。这是一个 OCTAL VALUE。格式：（用户）（组）（其他）其中每个组件是Read = 4，Write = 2和Execute = 1的总和。
                            例如，值0755转换为用户读写，Group和其他只读。默认值是0755.
                        -->
                        <!--string-->
                        <directoryMode/>
                        <!--
                            设置模块目录的绝对或相对位置。例如，“src / main / bin”会选择定义这个依赖关系的项目的这个子目录。
                        -->
                        <!--string-->
                        <directory/>
                        <!--
                            设置此文件集中文件的行结束符。有效值：
                            “keep” - 保留所有的行结束
                            “unix” - 使用Unix风格的行尾（即“\ n”）
                            “lf” - 使用一个换行符结束符（即“\ n”）
                            “dos” - 使用DOS / Windows风格的行尾（即“\ r \ n”）
                            “windows” - 使用DOS / Windows风格的行尾（即“\ r \ n”）
                            “crlf” - 使用回车，换行符结尾（即“\ r \ n”）
                        -->
                        <!--string-->
                        <lineEnding/>
                        <!--
                            是否在复制文件时过滤符号，使用构建配置中的属性。（从2.2-beta-1开始）
                            默认值是：false。
                        -->
                        <!--boolean-->
                        <filtered/>
                    </fileSet>
                </fileSets>
                <!--
                    指定模块的finalName是否应该添加到应用于它的任何fileSets的outputDirectory值。（从2.2-beta-1开始）
                    默认值是：true。
                -->
                <!--boolean-->
                <includeModuleDirectory/>
                <!--
                    指定是否应从应用于该模块的文件集中排除当前模块下方的子模块目录。如果仅仅意味着复制与此ModuleSet匹配的确切模块列表的源，忽略（或单独处理）当前目录下目录中存在的模块，这可能会很有用。（从2.2-beta-1开始）
                    默认值是：true。
                -->
                <!--boolean-->
                <excludeSubModuleDirectories/>
                <!--
                    设置此程序集中包含的所有模块基本目录的映射模式。注意：只有在includeModuleDirectory == true的情况下才会使用此字段。
                    缺省值是在 2.2-beta-1中是$ {artifactId}，以及后续版本中是$ {module.artifactId}。（从2.2-beta-1开始）
                    默认值是：$ {module.artifactId}。
                -->
                <!--string-->
                <outputDirectoryMapping/>
            </sources>
            <!--
                    如果存在，插件将在生成的程序集中包含来自该组的所包含模块的二进制文件。
                    包含用于将项目模块的二进制文件包含在程序集中的配置选项。
            -->
            <!--ModuleBinaries-->
            <binaries>
                <!--
                    设置输出目录相对于程序集根目录的根目录。例如，“log”会将指定的文件放在归档根目录下的日志目录中。
                -->
                <!--string-->
                <outputDirectory/>
                <!--
                    （许多） 当存在<include>子元素时，它们定义一组要包含的工件坐标。如果不存在，则<includes>表示所有有效值。
                    工件坐标可以以简单的groupId：artifactId形式给出，或者可以以groupId：artifactId：type [：classifier]：version的形式完全限定。
                    另外，可以使用通配符，如*：maven- *
                -->
                <!--List<String>-->
                <includes/>
                <!--
                    （许多） 当存在<exclude>子元素时，它们定义一组依赖项工件坐标以排除。如果不存在，则<excludes>不表示排除。
                    工件坐标可以以简单的groupId：artifactId形式给出，或者可以以groupId：artifactId：type [：classifier]：version的形式完全限定。
                    另外，可以使用通配符，如*：maven- *
                -->
                <!--List<String>-->
                <excludes/>
                <!--
                    与UNIX权限类似，设置所包含文件的文件模式。这是一个 OCTAL VALUE。格式：（用户）（组）（其他）其中每个组件是Read = 4，Write = 2和Execute = 1的总和。
                    例如，值0644转换为用户读写，组和其他只读。默认值是0644
                -->
                <!--string-->
                <fileMode/>
                <!--
                    与UNIX权限类似，设置包含的目录的目录模式。这是一个 OCTAL VALUE。格式：（用户）（组）（其他）[Format: (User)(Group)(Other) ] 其中每个组件是Read = 4，Write = 2和Execute = 1的总和。
                    例如，值0755转换为用户读写，Group和其他只读。默认值是0755.
                -->
                <!--string-->
                <directoryMode/>
                <!--
                    指定时，attachmentClassifier将使汇编器查看附加到模块的工件，而不是主工程工件。如果能够找到与指定分类符匹配的附件，则会使用它; 否则，会抛出异常。（从2.2-beta-1开始）
                -->
                <!--string-->
                <attachmentClassifier/>
                <!--
                    如果设置为true，插件将包含这里包含的项目模块的直接和传递依赖关系。否则，它将只包含模块包。
                    默认值是：true。
                -->
                <!--boolean-->
                <includeDependencies/>
                <!--List<DependencySet>-->
                <dependencySets>
                    <!--
                        依赖关系集允许在程序集中包含和排除项目依赖关系。
                    -->
                    <dependencySet>
                        <!--
                                设置输出目录相对于程序集根目录的根目录。例如，“log”会将指定的文件放在归档根目录下的日志目录中。
                            -->
                        <!--string-->
                        <outputDirectory/>
                        <!--
                            （许多） 当存在<include>子元素时，它们定义一组要包含的工件坐标。如果不存在，则<includes>表示所有有效值。
                            工件坐标可以以简单的groupId：artifactId形式给出，或者可以以groupId：artifactId：type [：classifier]：version的形式完全限定。
                            另外，可以使用通配符，如*：maven- *
                        -->
                        <!--List<String>-->
                        <includes/>
                        <!--
                            （许多） 当存在<exclude>子元素时，它们定义一组依赖项工件坐标以排除。如果不存在，则<excludes>不表示排除。
                            工件坐标可以以简单的groupId：artifactId形式给出，或者可以以groupId：artifactId：type [：classifier]：version的形式完全限定。
                            另外，可以使用通配符，如*：maven- *
                        -->
                        <!--List<String>-->
                        <excludes/>
                        <!--
                            与UNIX权限类似，设置所包含文件的文件模式。这是一个 OCTAL VALUE。格式：（用户）（组）（其他）其中每个组件是Read = 4，Write = 2和Execute = 1的总和。
                            例如，值0644转换为用户读写，组和其他只读。默认值是0644
                        -->
                        <!--string-->
                        <fileMode/>
                        <!--
                            与UNIX权限类似，设置包含的目录的目录模式。这是一个 OCTAL VALUE。格式：（用户）（组）（其他）[Format: (User)(Group)(Other) ] 其中每个组件是Read = 4，Write = 2和Execute = 1的总和。
                            例如，值0755转换为用户读写，Group和其他只读。默认值是0755.
                        -->
                        <!--string-->
                        <directoryMode/>
                        <!--
                            如果指定为true，那么在程序集创建过程中任何用于过滤实际构件的包含/排除模式都将导致构建失败，并显示错误。这是为了强调过时的包含或排除，或者表示程序集描述符配置不正确。（从2.2开始）
                            默认值是：false。
                        -->
                        <!--boolean-->
                        <useStrictFiltering/>
                        <!--
                            为此程序集中包含的所有依赖项设置映射模式。（从2.2-beta-2开始； 2.2-beta-1使用$ {artifactId} - $ {version} $ {dashClassifier？}。$ {extension}作为默认值）。
                            默认值是：$ {artifact.artifactId} - $ {artifact.version} $ {dashClassifier？}。$ {artifact.extension}。
                        -->
                        <!--string-->
                        <outputFileNameMapping/>
                        <!--
                            如果设置为true，则此属性将所有依赖项解包到指定的输出目录中。设置为false时，依赖关系将被包含为档案（jar）。只能解压jar，zip，tar.gz和tar.bz压缩文件。
                            默认值是：false。
                        -->
                        <!--boolean-->
                        <unpack/>
                        <!--
                            允许指定包含和排除以及过滤选项，以指定从相关性工件解压缩的项目。（从2.2-beta-1开始）
                        -->
                        <unpackOptions>
                            <!--
                                （许多） 文件和/或目录模式的集合，用于匹配将在解压缩时从归档文件中包含的项目。每个项目被指定为<include> some / path </ include>（从2.2-beta-1开始）
                            -->
                            <!--List<String>-->
                            <includes/>
                            <!--
                                （许多） 用于匹配项目的文件和/或目录模式的集合，在解压缩时将其从归档文件中排除。每个项目被指定为<exclude> some / path </ exclude>（从2.2-beta-1开始）
                            -->
                            <!--List<String>-->
                            <excludes/>
                            <!--
                                是否使用构建配置中的属性过滤从档案中解压缩的文件中的符号。（从2.2-beta-1开始）
                                默认值是：false。
                            -->
                            <!--boolean-->
                            <filtered/>
                            <!--
                                设置文件的行尾。（从2.2开始）有效值：
                                “keep” - 保留所有的行结束
                                “unix” - 使用Unix风格的行结尾
                                “lf” - 使用单个换行符结束符
                                “dos” - 使用DOS风格的行尾
                                “ crlf ” - 使用Carraige返回，换行符结束
                            -->
                            <!--string-->
                            <lineEnding/>
                            <!--
                                在计算受该集合影响的文件时，是否应该使用标准排除模式，例如那些匹配CVS和Subversion元数据文件的排除模式。为了向后兼容，默认值是true。（从2.2开始）
                                默认值是：true。
                            -->
                            <!--boolean-->
                            <useDefaultExcludes/>
                            <!--
                                允许指定解压档案时使用的编码，支持指定编码的unarchiver。如果未指定，将使用归档程序默认值。Archiver默认值通常代表理智（modern）的values。
                            -->
                            <!--string-->
                            <encoding/>
                        </unpackOptions>
                        <!--
                            为此dependencySet设置依赖项范围。
                            默认值是：runtime。
                        -->
                        <!--string-->
                        <scope/>
                        <!--
                            确定当前项目构建过程中产生的工件是否应该包含在这个依赖集中。（从2.2-beta-1开始）
                            默认值是：true。
                        -->
                        <!--boolean-->
                        <useProjectArtifact/>
                        <!--
                            确定当前项目构建过程中产生的附件是否应该包含在这个依赖集中。（从2.2-beta-1开始）
                            默认值是：false。
                        -->
                        <!--boolean-->
                        <useProjectAttachments/>
                        <!--
                            确定是否将传递依赖项包含在当前依赖项集的处理中。如果为true，那么include / excludes / useTransitiveFiltering将应用于传递依赖项构件以及主项目依赖项构件。
                            如果为false，则useTransitiveFiltering无意义，并且包含/排除仅影响项目的直接依赖关系。
                            默认情况下，这个值是真的。（从2.2-beta-1开始）
                            默认值是：true。
                        -->
                        <!--boolean-->
                        <useTransitiveDependencies/>
                        <!--
                            确定此依赖项集中的包含/排除模式是否将应用于给定工件的传递路径。
                            如果为真，并且当前工件是由包含或排除模式匹配的另一个工件引入的传递依赖性，则当前工件具有与其相同的包含/排除逻辑。
                            默认情况下，此值为false，以保持与2.1版的向后兼容性。这意味着包含/排除仅仅直接应用于当前的工件，而不应用于传入的工件。（从2.2-beta-1）
                            默认值为：false。
                        -->
                        <!--boolean-->
                        <useTransitiveFiltering/>
                    </dependencySet>
                </dependencySets>
                <!--
                    如果设置为true，则此属性将所有模块包解包到指定的输出目录中。当设置为false时，模块包将作为归档（jar）包含在内。
                    默认值是：true。
                -->
                <!--boolean-->
                <unpack/>
                <!--
                    允许指定包含和排除以及过滤选项，以指定从相关性工件解压缩的项目。（从2.2-beta-1开始）
                -->
                <unpackOptions>
                    <!--
                        （许多） 文件和/或目录模式的集合，用于匹配将在解压缩时从归档文件中包含的项目。每个项目被指定为<include> some / path </ include>（从2.2-beta-1开始）
                    -->
                    <!--List<String>-->
                    <includes/>
                    <!--
                        （许多） 用于匹配项目的文件和/或目录模式的集合，在解压缩时将其从归档文件中排除。每个项目被指定为<exclude> some / path </ exclude>（从2.2-beta-1开始）
                    -->
                    <!--List<String>-->
                    <excludes/>
                    <!--
                        是否使用构建配置中的属性过滤从档案中解压缩的文件中的符号。（从2.2-beta-1开始）
                        默认值是：false。
                    -->
                    <!--boolean-->
                    <filtered/>
                    <!--
                        设置文件的行尾。（从2.2开始）有效值：
                        “keep” - 保留所有的行结束
                        “unix” - 使用Unix风格的行结尾
                        “lf” - 使用单个换行符结束符
                        “dos” - 使用DOS风格的行尾
                        “ crlf ” - 使用Carraige返回，换行符结束
                    -->
                    <!--string-->
                    <lineEnding/>
                    <!--
                        在计算受该集合影响的文件时，是否应该使用标准排除模式，例如那些匹配CVS和Subversion元数据文件的排除模式。为了向后兼容，默认值是true。（从2.2开始）
                        默认值是：true。
                    -->
                    <!--boolean-->
                    <useDefaultExcludes/>
                    <!--
                        允许指定解压档案时使用的编码，支持指定编码的unarchiver。如果未指定，将使用归档程序默认值。Archiver默认值通常代表理智（modern）的values。
                    -->
                    <!--string-->
                    <encoding/>
                </unpackOptions>
                <!--
                    设置此程序集中包含的所有非UNPACKED依赖关系的映射模式。（由于2.2-beta-2; 2.2-beta-1使用$ {artifactId} - $ {version} $ {dashClassifier？}。$ {extension}作为默认值）注意：如果dependencySet指定unpack == true，则outputFileNameMapping将不要使用; 在这些情况下，使用outputDirectory。有关可用于outputFileNameMapping参数的条目的更多详细信息，请参阅插件FAQ。
                    默认值是：$ {module.artifactId} - $ {module.version} $ {dashClassifier？}。$ {module.extension}。
                -->
                <!--string-->
                <outputFileNameMapping/>
            </binaries>
        </moduleSet>
    </moduleSets>
    <!--
        （许多） 指定在程序集中包含哪些文件组。fileSet通过提供一个或多个<fileSet>子元素来指定。
    -->
    <!--List<FileSet>-->
    <fileSets>
        <!--
            fileSet允许将文件组包含到程序集中。
        -->
        <fileSet>
            <!--
                在计算受该集合影响的文件时，是否应该使用标准排除模式，例如那些匹配CVS和Subversion元数据文件的排除模式。为了向后兼容，默认值是true。（从2.2-beta-1开始）
                默认值是：true。
            -->
            <!--boolean-->
            <useDefaultExcludes/>
            <!--
                设置输出目录相对于程序集根目录的根目录。例如，“日志”将把指定的文件放在日志目录中。
            -->
            <!--string-->
            <outputDirectory/>
            <!--
                （许多） 当<include>子元素存在时，它们定义一组要包含的文件和目录。如果不存在，则<includes>表示所有有效值。
            -->
            <!--List<String>-->
            <includes/>
            <!--
                （许多） 当存在<exclude>子元素时，它们定义一组要排除的文件和目录。如果不存在，则<excludes>不表示排除。
            -->
            <!--List<String>-->
            <excludes/>
            <!--
                与UNIX权限类似，设置所包含文件的文件模式。这是一个 OCTAL VALUE。格式：（用户）（组）（其他）其中每个组件是Read = 4，Write = 2和Execute = 1的总和。
                例如，值0644转换为用户读写，组和其他只读。默认值是0644.
            -->
            <!--string-->
            <fileMode/>
            <!--
                与UNIX权限类似，设置包含的目录的目录模式。这是一个 OCTAL VALUE。格式：（用户）（组）（其他）其中每个组件是Read = 4，Write = 2和Execute = 1的总和。
                例如，值0755转换为用户读写，Group和其他只读。默认值是0755.
            -->
            <!--string-->
            <directoryMode/>
            <!--
                设置模块目录的绝对或相对位置。例如，“src / main / bin”会选择定义这个依赖关系的项目的这个子目录。
            -->
            <!--string-->
            <directory/>
            <!--
                设置此文件集中文件的行结束符。有效值：
                “keep” - 保留所有的行结束
                “unix” - 使用Unix风格的行尾（即“\ n”）
                “lf” - 使用一个换行符结束符（即“\ n”）
                “dos” - 使用DOS / Windows风格的行尾（即“\ r \ n”）
                “windows” - 使用DOS / Windows风格的行尾（即“\ r \ n”）
                “crlf” - 使用回车，换行符结尾（即“\ r \ n”）
            -->
            <!--string-->
            <lineEnding/>
            <!--
                是否在复制文件时过滤符号，使用构建配置中的属性。（从2.2-beta-1开始）
                默认值是：false。
            -->
            <!--boolean-->
            <filtered/>
        </fileSet>
    </fileSets>
    <!--
        （许多） 指定在程序集中包含哪些单个文件。通过提供一个或多个<file>子元素来指定文件。
    -->
    <!--List<FileItem>-->
    <files>
        <!--
            一个文件允许单个文件包含选项来更改不受fileSets支持的目标文件名。
        -->
        <file>
            <!--
                设置要包含在程序集中的文件的模块目录的绝对路径或相对路径。
            -->
            <!--string-->
            <source/>
            <!--
                设置输出目录相对于程序集根目录的根目录。例如，“日志”将把指定的文件放在日志目录中。
            -->
            <!--string-->
            <outputDirectory/>
            <!--
                在outputDirectory中设置目标文件名。默认是与源文件相同的名称。
            -->
            <!--string-->
            <destName/>
            <!--
                与UNIX权限类似，设置所包含文件的文件模式。这是一个八卦价值。格式：（用户）（组）（其他）其中每个组件是Read = 4，Write = 2和Execute = 1的总和。
                例如，值0644转换为用户读写，组和其他只读。默认值是0644
            -->
            <!--string-->
            <fileMode/>
            <!--
                设置此文件中文件的行结束符。有效值是：
                “keep” - 保留所有的行结束
                “unix” - 使用Unix风格的行尾（即“\ n”）
                “lf” - 使用一个换行符结束符（即“\ n”）
                “dos” - 使用DOS / Windows风格的行尾（即“\ r \ n”）
                “windows” - 使用DOS / Windows风格的行尾（即“\ r \ n”）
                “crlf” - 使用回车，换行符结尾（即“\ r \ n”）
            -->
            <!--string-->
            <lineEnding/>
            <!--
                设置是否确定文件是否被过滤。
                默认值是：false。
            -->
            <!--boolean-->
            <filtered/>
        </file>
    </files>
    <!--List<DependencySet>-->
    <dependencySets>
        <!--
            依赖关系集允许在程序集中包含和排除项目依赖关系。
        -->
        <dependencySet>
            <!--
                    设置输出目录相对于程序集根目录的根目录。例如，“log”会将指定的文件放在归档根目录下的日志目录中。
                -->
            <!--string-->
            <outputDirectory/>
            <!--
                （许多） 当存在<include>子元素时，它们定义一组要包含的工件坐标。如果不存在，则<includes>表示所有有效值。
                工件坐标可以以简单的groupId：artifactId形式给出，或者可以以groupId：artifactId：type [：classifier]：version的形式完全限定。
                另外，可以使用通配符，如*：maven- *
            -->
            <!--List<String>-->
            <includes/>
            <!--
                （许多） 当存在<exclude>子元素时，它们定义一组依赖项工件坐标以排除。如果不存在，则<excludes>不表示排除。
                工件坐标可以以简单的groupId：artifactId形式给出，或者可以以groupId：artifactId：type [：classifier]：version的形式完全限定。
                另外，可以使用通配符，如*：maven- *
            -->
            <!--List<String>-->
            <excludes/>
            <!--
                与UNIX权限类似，设置所包含文件的文件模式。这是一个 OCTAL VALUE。格式：（用户）（组）（其他）其中每个组件是Read = 4，Write = 2和Execute = 1的总和。
                例如，值0644转换为用户读写，组和其他只读。默认值是0644
            -->
            <!--string-->
            <fileMode/>
            <!--
                与UNIX权限类似，设置包含的目录的目录模式。这是一个 OCTAL VALUE。格式：（用户）（组）（其他）[Format: (User)(Group)(Other) ] 其中每个组件是Read = 4，Write = 2和Execute = 1的总和。
                例如，值0755转换为用户读写，Group和其他只读。默认值是0755.
            -->
            <!--string-->
            <directoryMode/>
            <!--
                如果指定为true，那么在程序集创建过程中任何用于过滤实际构件的包含/排除模式都将导致构建失败，并显示错误。这是为了强调过时的包含或排除，或者表示程序集描述符配置不正确。（从2.2开始）
                默认值是：false。
            -->
            <!--boolean-->
            <useStrictFiltering/>
            <!--
                为此程序集中包含的所有依赖项设置映射模式。（从2.2-beta-2开始； 2.2-beta-1使用$ {artifactId} - $ {version} $ {dashClassifier？}。$ {extension}作为默认值）。
                默认值是：$ {artifact.artifactId} - $ {artifact.version} $ {dashClassifier？}。$ {artifact.extension}。
            -->
            <!--string-->
            <outputFileNameMapping/>
            <!--
                如果设置为true，则此属性将所有依赖项解包到指定的输出目录中。设置为false时，依赖关系将被包含为档案（jar）。只能解压jar，zip，tar.gz和tar.bz压缩文件。
                默认值是：false。
            -->
            <!--boolean-->
            <unpack/>
            <!--
                允许指定包含和排除以及过滤选项，以指定从相关性工件解压缩的项目。（从2.2-beta-1开始）
            -->
            <unpackOptions>
                <!--
                    （许多） 文件和/或目录模式的集合，用于匹配将在解压缩时从归档文件中包含的项目。每个项目被指定为<include> some / path </ include>（从2.2-beta-1开始）
                -->
                <!--List<String>-->
                <includes/>
                <!--
                    （许多） 用于匹配项目的文件和/或目录模式的集合，在解压缩时将其从归档文件中排除。每个项目被指定为<exclude> some / path </ exclude>（从2.2-beta-1开始）
                -->
                <!--List<String>-->
                <excludes/>
                <!--
                    是否使用构建配置中的属性过滤从档案中解压缩的文件中的符号。（从2.2-beta-1开始）
                    默认值是：false。
                -->
                <!--boolean-->
                <filtered/>
                <!--
                    设置文件的行尾。（从2.2开始）有效值：
                    “keep” - 保留所有的行结束
                    “unix” - 使用Unix风格的行结尾
                    “lf” - 使用单个换行符结束符
                    “dos” - 使用DOS风格的行尾
                    “crlf ” - 使用Carraige返回，换行符结束
                -->
                <!--string-->
                <lineEnding/>
                <!--
                    在计算受该集合影响的文件时，是否应该使用标准排除模式，例如那些匹配CVS和Subversion元数据文件的排除模式。为了向后兼容，默认值是true。（从2.2开始）
                    默认值是：true。
                -->
                <!--boolean-->
                <useDefaultExcludes/>
                <!--
                    允许指定解压档案时使用的编码，支持指定编码的unarchiver。如果未指定，将使用归档程序默认值。Archiver默认值通常代表理智（modern）的values。
                -->
                <!--string-->
                <encoding/>
            </unpackOptions>
            <!--
                为此dependencySet设置依赖项范围。
                默认值是：runtime。
            -->
            <!--string-->
            <scope/>
            <!--
                确定当前项目构建过程中产生的工件是否应该包含在这个依赖集中。（从2.2-beta-1开始）
                默认值是：true。
            -->
            <!--boolean-->
            <useProjectArtifact/>
            <!--
                确定当前项目构建过程中产生的附件是否应该包含在这个依赖集中。（从2.2-beta-1开始）
                默认值是：false。
            -->
            <!--boolean-->
            <useProjectAttachments/>
            <!--
                确定是否将传递依赖项包含在当前依赖项集的处理中。如果为true，那么include / excludes / useTransitiveFiltering将应用于传递依赖项构件以及主项目依赖项构件。
                如果为false，则useTransitiveFiltering无意义，并且包含/排除仅影响项目的直接依赖关系。
                默认情况下，这个值是真的。（从2.2-beta-1开始）
                默认值是：true。
            -->
            <!--boolean-->
            <useTransitiveDependencies/>
            <!--
                确定此依赖项集中的包含/排除模式是否将应用于给定工件的传递路径。
                如果为真，并且当前工件是由包含或排除模式匹配的另一个工件引入的传递依赖性，则当前工件具有与其相同的包含/排除逻辑。
                默认情况下，此值为false，以保持与2.1版的向后兼容性。这意味着包含/排除仅仅直接应用于当前的工件，而不应用于传入的工件。（从2.2-beta-1）
                默认值为：false。
            -->
            <!--boolean-->
            <useTransitiveFiltering/>
        </dependencySet>
    </dependencySets>
    <!--
        定义要包含在程序集中的Maven仓库。可用于存储库中的工件是项目的依赖工件。创建的存储库包含所需的元数据条目，并且还包含sha1和md5校验和。这对创建将被部署到内部存储库的档案很有用。
        注意：目前，只有来自中央存储库的工件才被允许。
    -->
    <!--List<Repository>-->
    <repositories>
        <repository>
            <!--
                设置输出目录相对于程序集根目录的根目录。例如，“log”会将指定的文件放在归档根目录下的日志目录中。
            -->
            <!--string-->
            <outputDirectory/>
            <!--
                （许多） 当存在<include>子元素时，它们定义一组包含的项目坐标。如果不存在，则<includes>表示所有有效值。
                工件坐标可以以简单的groupId：artifactId形式给出，或者可以以groupId：artifactId：type [：classifier]：version的形式完全限定。
                另外，可以使用通配符，如*：maven- *
            -->
            <!--List<String>-->
            <includes/>
            <!--
                （许多） 当存在<exclude>子元素时，它们定义一组要排除的项目工件坐标。如果不存在，则<excludes>不表示排除。
                工件坐标可以以简单的groupId：artifactId形式给出，或者可以以groupId：artifactId：type [：classifier]：version的形式完全限定。
                另外，可以使用通配符，如*：maven- *
            -->
            <!--List<String>-->
            <excludes/>
            <!--
                    与UNIX权限类似，设置所包含文件的文件模式。这是一个 OCTAL VALUE。格式：（用户）（组）（其他）其中每个组件是Read = 4，Write = 2和Execute = 1的总和。
                    例如，值0644转换为用户读写，组和其他只读。默认值是0644
                -->
            <!--string-->
            <fileMode/>
            <!--
                与UNIX权限类似，设置包含的目录的目录模式。这是一个 OCTAL VALUE。格式：（用户）（组）（其他）[Format: (User)(Group)(Other) ] 其中每个组件是Read = 4，Write = 2和Execute = 1的总和。
                例如，值0755转换为用户读写，Group和其他只读。默认值是0755.
            -->
            <!--string-->
            <directoryMode/>
            <!--
                如果设置为true，则此属性将触发创建存储库元数据，这将允许存储库用作功能性远程存储库。
                默认值是：false。
            -->
            <!--boolean-->
            <includeMetadata/>
            <!--
                （许多） 指定要将一组工件与指定的版本对齐。groupVersionAlignment通过提供一个或多个<groupVersionAlignment>子元素来指定。
                允许一组工件与指定的版本对齐。
            -->
            <!--List<GroupVersionAlignment>-->
            <groupVersionAlignments>
                <groupVersionAlignment>
                    <!--
                        要为其对齐版本的工件的groupId。
                    -->
                    <!--string-->
                    <id/>
                    <!--
                        您想要将该组对齐的版本。
                    -->
                    <!--string-->
                    <version/>
                    <!--
                        （许多） 当存在<exclude>子元素时，它们定义要排除的构件的artifactIds。如果不存在，则<excludes>不表示排除。排除是通过提供一个或多个<exclude>子元素来指定的。
                    -->
                    <!--List<String>-->
                    <excludes/>
                </groupVersionAlignment>
            </groupVersionAlignments>
            <!--
                指定此存储库中包含的工件的范围。（从2.2-beta-1开始）
                默认值是：runtime。
            -->
            <!--string-->
            <scope/>
        </repository>
    </repositories>
    <!--
        （许多） 指定要包含在程序集中的共享组件xml文件位置。指定的位置必须相对于描述符的基本位置。
        如果描述符是通过类路径中的<descriptorRef />元素找到的，那么它指定的任何组件也将在类路径中找到。
        如果通过路径名通过<descriptor />元素找到，则此处的值将被解释为相对于项目basedir的路径。
        当找到多个componentDescriptors时，它们的内容被合并。检查 描述符组件 了解更多信息。
        componentDescriptor通过提供一个或多个<componentDescriptor>子元素来指定。
    -->
    <!--List<String>-->
    <componentDescriptors/>
</assembly>
```

# Maven内置变量

* ${basedir} 项目根目录

* ${project.build.directory} 构建目录，缺省为target

* ${project.build.outputDirectory} 构建过程输出目录，缺省为target/classes

* \${project.build.finalName} 产出物名称，缺省为\${project.artifactId}-${project.version}

* ${project.packaging} 打包类型，缺省为jar

* ${project.xxx} 当前pom文件的任意节点的内容

# 依赖项的作用域

  在定义项目的依赖项的时候，我们可以通过scope来指定该依赖项的作用范围。scope的取值有compile、runtime、test、provided、system和import。

* **compile**：这是依赖项的默认作用范围，即当没有指定依赖项的scope时默认使用compile。compile范围内的依赖项在所有情况下都是有效的，包括运行、测试和编译时。
* **runtime**：表示该依赖项只有在运行时才是需要的，在编译的时候不需要。这种类型的依赖项将在运行和test的类路径下可以访问。
* **test**：表示该依赖项只对测试时有用，包括测试代码的编译和运行，对于正常的项目运行是没有影响的。
* **provided**：表示该依赖项将由JDK或者运行容器在运行时提供，也就是说由Maven提供的该依赖项我们只有在编译和测试时才会用到，而在运行时将由JDK或者运行容器提供。
* **system**：当scope为system时，表示该依赖项是我们自己提供的，不需要Maven到仓库里面去找。指定scope为system需要与另一个属性元素systemPath一起使用，它表示该依赖项在当前系统的位置，使用的是绝对路径。


# containerDescriptorHandler详解

Assembly插件支持常用的文件合并功能，特别是META-INF下的services文件或spirng文件(spring.handlers和spring.schemas)。

## metaInf-services

聚合所有的**META-INF/services**文件合并成一个文件

```html
<assembly xmlns="http://maven.apache.org/ASSEMBLY/2.0.0"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/ASSEMBLY/2.0.0 http://maven.apache.org/xsd/assembly-2.0.0.xsd">
  ....
  <containerDescriptorHandlers>
    <containerDescriptorHandler>
      <handlerName>metaInf-services</handlerName>
    </containerDescriptorHandler>
  </containerDescriptorHandlers>
</assembly>
```

## metaInf-spring

聚合所有的**META-INF/spring.**文件合并成一个文件

```html
<assembly xmlns="http://maven.apache.org/ASSEMBLY/2.0.0"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/ASSEMBLY/2.0.0 http://maven.apache.org/xsd/assembly-2.0.0.xsd">
  ....
  <containerDescriptorHandlers>
    <containerDescriptorHandler>
      <handlerName>metaInf-spring</handlerName>
    </containerDescriptorHandler>
  </containerDescriptorHandlers>
</assembly>
```

## file-aggregator

也可以给定正则表达式匹配文件合并成一个文件，以下匹配所有file.txt然后合并成一个file.txt

```html
<assembly xmlns="http://maven.apache.org/ASSEMBLY/2.0.0"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/ASSEMBLY/2.0.0 http://maven.apache.org/xsd/assembly-2.0.0.xsd">
  ....
  <containerDescriptorHandlers>
    <containerDescriptorHandler>
      <handlerName>file-aggregator</handlerName>
      <configuration>
        <filePattern>.*/file.txt</filePattern>
        <outputPath>file.txt</outputPath>
      </configuration>
    </containerDescriptorHandler>
  </containerDescriptorHandlers>
</assembly>
```

## **plexus**

该处理程序匹配每个`META-INF / plexus / components.xml`文件，并将它们聚合到一个有效的`META-INF / plexus / components.xml中`。

```xml
<assembly xmlns="http://maven.apache.org/ASSEMBLY/2.1.0"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/ASSEMBLY/2.1.0 http://maven.apache.org/xsd/assembly-2.1.0.xsd">
  ....
  <containerDescriptorHandlers>
    <containerDescriptorHandler>
      <handlerName>plexus</handlerName>
    </containerDescriptorHandler>
  </containerDescriptorHandlers>
</assembly>
```

### Custom container descriptor handlers

您可以通过创建实现`ContainerDescriptorHandler`的类来创建自己的容器描述符处理程序。作为示例，让我们创建一个处理程序，该处理程序会将配置的注释放在程序集描述符中配置的每个属性文件的前面。

我们首先使用以下POM 创建一个名为`custom-container-descriptor-handler`的新Maven项目：

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.test</groupId>
  <artifactId>custom-container-descriptor-handler</artifactId>
  <version>0.0.1-SNAPSHOT</version>
  <dependencies>
    <dependency>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-assembly-plugin</artifactId>
      <version>3.3.0</version>
    </dependency>
  </dependencies>
  <build>
    <plugins>
      <plugin>
        <groupId>org.codehaus.plexus</groupId>
        <artifactId>plexus-component-metadata</artifactId>
        <version>1.7.1</version>
        <executions>
          <execution>
            <goals>
              <goal>generate-metadata</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</project>
```

该POM声明了对Assembly Plugin的依赖关系，以便我们可以创建处理程序，并生成一个Plexus配置文件，以便可以在组装过程中通过依赖关系注入找到它。

实现`ContainerDescriptorHandler`需要定义几个方法：

* `isSelected`

  告诉是否应将在程序集描述符中配置的给定文件或目录添加到最终程序集中。典型的设置是防止添加某些文件，并让处理程序对它们进行处理。

* `getVirtualFiles`

  从程序集的根返回此处理程序将添加的每个文件的文件路径列表。

* `finalizeArchiveCreation`

  要创建程序集时调用的回调。此方法可用于添加处理程序对每个选定文件进行处理而生成的文件。**警告：**由于执行归档定稿的方式，我们需要在执行任何操作之前遍历归档中的每个资源。

* `finalizeArchiveExtraction`

  将程序集提取到目录中时调用的回调。此方法可用于处理由处理程序对每个选定文件的工作所产生的文件。

在每个属性文件前加注释的处理程序可能如下所示（此处使用Java 8功能）：

```java
package com.test;
 
import java.io.BufferedReader;
import java.io.BufferedWriter;
import java.io.File;
import java.io.IOException;
import java.io.InputStreamReader;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;
 
import org.apache.maven.plugins.assembly.filter.ContainerDescriptorHandler;
import org.apache.maven.plugins.assembly.utils.AssemblyFileUtils;
import org.codehaus.plexus.archiver.Archiver;
import org.codehaus.plexus.archiver.ArchiverException;
import org.codehaus.plexus.archiver.UnArchiver;
import org.codehaus.plexus.component.annotations.Component;
import org.codehaus.plexus.components.io.fileselectors.FileInfo;
 
//此处绑定handlerName
@Component(role = ContainerDescriptorHandler.class, hint = "custom")
public class MyCustomDescriptorHandler implements ContainerDescriptorHandler {
 
    private String comment;
 
    private Map<String, List<String>> catalog = new HashMap<>();
 
    private boolean excludeOverride = false;
 
    @Override
    public void finalizeArchiveCreation(Archiver archiver) throws ArchiverException {
        archiver.getResources().forEachRemaining(a -> {}); // necessary to prompt the isSelected() call
 
        for (Map.Entry<String, List<String>> entry : catalog.entrySet())
        {
            String name = entry.getKey();
            String fname = new File(name).getName();
 
            Path p;
            try {
                p = Files.createTempFile("assembly-" + fname, ".tmp");
            } catch (IOException e) {
                throw new ArchiverException("Cannot create temporary file to finalize archive creation", e);
            }
 
            try (BufferedWriter writer = Files.newBufferedWriter(p, StandardCharsets.ISO_8859_1)) {
                writer.write("# " + comment);
                for (String line : entry.getValue()) {
                    writer.newLine();
                    writer.write(line);
                }
            } catch (IOException e) {
                throw new ArchiverException("Error adding content of " + fname + " to finalize archive creation", e);
            }
 
            File file = p.toFile();
            file.deleteOnExit();
            excludeOverride = true;
            archiver.addFile(file, name);
            excludeOverride = false;
        }
    }
 
    @Override
    public void finalizeArchiveExtraction(UnArchiver unarchiver) throws ArchiverException { }
 
    @Override
    public List<String> getVirtualFiles() {
        return new ArrayList<>(catalog.keySet());
    }
 
    @Override
    public boolean isSelected(FileInfo fileInfo) throws IOException {
        if (excludeOverride) {
            return true;
        }
        String name = AssemblyFileUtils.normalizeFileInfo(fileInfo);
        if (fileInfo.isFile() && AssemblyFileUtils.isPropertyFile(name)) {
            catalog.put(name, readLines(fileInfo));
            return false;
        }
        return true;
    }
 
    private List<String> readLines(FileInfo fileInfo) throws IOException {
        try (BufferedReader reader = new BufferedReader(new InputStreamReader(fileInfo.getContents(), StandardCharsets.ISO_8859_1))) {
            return reader.lines().collect(Collectors.toList());
        }
    }
 
    public void setComment(String comment) {
        this.comment = comment;
    }
 
}
```

它是一个Plexus组件，具有`ContainerDescriptorHandler`角色，其`提示`为`custom`，从而与其他处理程序区分开。

它选择每个属性文件并将其内容存储到目录映射中，其中键是文件名，值是其行的列表。那些匹配的文件不会添加到程序集中，因为处理程序需要先对其进行处理。在程序集创建期间，它将创建临时文件，其内容为先前读取的行，并带有自定义注释。然后将它们以其先前的名称重新添加到存档中。请注意，此简单处理程序不会聚合具有相同名称的文件-可以对其进行增强。将临时文件添加到存档后，将自动调用`isSelected`方法，因此我们需要将布尔`excludeOverride`设置为`true`，以确保未完成目录处理部分。

最后一个因素是在某些Maven项目的程序集描述符中使用我们的自定义处理程序。假设此项目中有一个`src / samples`目录，其中包含一个名为`test.xml`的XML文件和一个名为`test.properties`的属性文件。具有以下描述符格式

```xml
<assembly xmlns="http://maven.apache.org/ASSEMBLY/2.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/ASSEMBLY/2.0.0 http://maven.apache.org/xsd/assembly-2.0.0.xsd">
  <id>dist</id>
  <formats>
    <format>zip</format>
  </formats>
  <containerDescriptorHandlers>
    <containerDescriptorHandler>
        <!--绑定自定义的handler，与hint相同 -->
      <handlerName>custom</handlerName>
      <configuration>
        <comment>A comment</comment>
      </configuration>
    </containerDescriptorHandler>
  </containerDescriptorHandlers>
  <fileSets>
    <fileSet>
      <directory>src/samples</directory>
      <outputDirectory></outputDirectory>
    </fileSet>
  </fileSets>
</assembly>
```

和以下程序集插件配置

```xml
<project>
  [...]
  <build>
    [...]
    <plugins>
      [...]
      <plugin>
        <artifactId>maven-assembly-plugin</artifactId>
        <version>3.3.0</version>
        <executions>
          <execution>
            <phase>package</phase>
            <goals>
              <goal>single</goal>
            </goals>
            <configuration>
              <descriptors>
                <descriptor>src/assemble/assembly.xml</descriptor>
              </descriptors>
            </configuration>
          </execution>
        </executions>
          <!--引入自定义的handler -->
        <dependencies>
          <dependency>
            <groupId>com.test</groupId>
            <artifactId>custom-container-descriptor-handler</artifactId>
            <version>0.0.1-SNAPSHOT</version>
          </dependency>
        </dependencies>
      </plugin>
  [...]
</project>
```

生成的程序集将在基本目录下包含`test.xml`和`test.properties`，仅后者以`＃A ``comment`开头。