---
title: "Java 8"
nav-parent_id: api-concepts
nav-pos: 20
---
<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

Java8引入了一些语言的新特性，使编程变得更加简洁和快速.其中有一个最为重要的特性,被称作"Lambda表达式", Java8打开了函数式编程的大门. Lambda表达式使实现和传递函数都变得更加直接，同时不需要重新定义新增的(匿名)类.

最新版本的Flink支持所有Java API中Lambda表达式的操作.这篇文档展示如何使用Lambda表达式以及表述当前的限制. 查看通用的Flink API的介绍,请参照 [应用开发 指南]({{ site.baseurl }}/dev/api_concepts.html)

* TOC
{:toc}

### 示例

如下的示例阐述了如何实现一个简单的内联的`map()`函数，Lambd函数进行开平方.`map()`函数的输入参数和输出参数不需要事先定义类型，它们会通过Java 8编译器推断出来.

~~~java
env.fromElements(1, 2, 3)
// returns the squared i
.map(i -> i*i)
.print();
~~~

接下来的两个例子展现了使用`Collector`做输出的不同实现方式.像`flatMap()`这样的函数，需要一个输出参数(这个例子中是`String`) ，需要事先定义好使 `Collector`是类型安全的.
如何`Collector`类型不能通过 环境的上下文推断出来,它需要在Lambda表达式的参数列表中手动的事先定义好.否则它的输出参数会被当成`Object`类型，这会导致不可预料的行为.

~~~java
DataSet<Integer> input = env.fromElements(1, 2, 3);

// collector type must be declared
input.flatMap((Integer number, Collector<String> out) -> {
    StringBuilder builder = new StringBuilder();
    for(int i = 0; i < number; i++) {
        builder.append("a");
        out.collect(builder.toString());
    }
})
// returns (on separate lines) "a", "a", "aa", "a", "aa", "aaa"
.print();
~~~

~~~java
DataSet<Integer> input = env.fromElements(1, 2, 3);

// collector type must not be declared, it is inferred from the type of the dataset
DataSet<String> manyALetters = input.flatMap((number, out) -> {
    StringBuilder builder = new StringBuilder();
    for(int i = 0; i < number; i++) {
       builder.append("a");
       out.collect(builder.toString());
    }
});

// returns (on separate lines) "a", "a", "aa", "a", "aa", "aaa"
manyALetters.print();
~~~

下面的代码展示了一个word count程序，它使用了大量的Lambda表达式.

~~~java
DataSet<String> input = env.fromElements("Please count", "the words", "but not this");

// filter out strings that contain "not"
input.filter(line -> !line.contains("not"))
// split each line by space
.map(line -> line.split(" "))
// emit a pair <word,1> for each array element
.flatMap((String[] wordArray, Collector<Tuple2<String, Integer>> out)
    -> Arrays.stream(wordArray).forEach(t -> out.collect(new Tuple2<>(t, 1)))
    )
// group and sum up
.groupBy(0).sum(1)
// print
.print();
~~~

### 编译器限制
当前,Flink仅在以下条件中可以完美支持Lambda表达式，它们使用**Eclipse Luna 4.4.2 (以及以上版本)的 JDT 编译器 **编译.

只有Eclipse JDT 编译器保留了必要的原始类型使整个Lambda表达式是类型安全的.其他编译器例如OpenJDK和Oracle JDK的`javac` ，抛弃了Lambda表达式中所有的的原始类型. 它意味着诸如`Tuple2<String, Integer>` 或者 `Collector<String>`这样的类型，在Lambda表达式中原本作为入参或者出参的参数，最后在编译后的 `.class` 文件中被转化成`Tuple2` 或者 `Collector`这样的类型, 这对于Flink 编译器来讲，信息太少了.

如何使用JDT编译器编译带有Lambda表达式的Flink任务将作为下一章节的内容.

然而, 使用除了Eclipse JDT之外的Java 8的编译器来实现 `map()`或者`filter()`这样的Lambda表达式是可行的， 只要函数里没有 `Collector`或者 `Iterable`， *并且* 仅当函数使用例如 `Integer`, `Long`, `String`, `MyOwnClass`这样的非参数化类型 (非泛型!).

#### 使用Eclipse JDT 编译器和Maven编译Flink任务

如果你使用Eclipse IDE, 在经过一些配置后就可以在IDE里面运行和调试Flink代码. Eclipse IDE默认使用Eclipse JDT编译器编译Java代码. 接下来的内容描述如何配置Eclipse IDE.

如果你使用另外的IDE，例如IntelliJ IDEA。或者你想要通过Maven打包Jar文件部署到集群上运行自己的任务, 那么你需要修改工程下面的`pom.xml` 文件然后使用Maven构建自己的程序. [quickstart]({{site.baseurl}}/quickstart/setup_quickstart.html) 包含了一些预先配置好的Maven 工程，可以用来做参考. 如果想使用Java8 Lambda表达式，在生成的`pom.xml`中取消掉被提及到的注释.

另一种方式, 你可以手动的在 Maven`pom.xml`文件中添加下面几行配置. Maven会使用the Eclipse JDT编译器编译.

~~~xml
<!-- 将以下几行添加到pom.xml的"project/build/pluginManagement/plugins"之间 -->

<plugin>
    <!-- Use compiler plugin with tycho as the adapter to the JDT compiler. -->
    <artifactId>maven-compiler-plugin</artifactId>
    <configuration>
        <source>1.8</source>
        <target>1.8</target>
        <compilerId>jdt</compilerId>
    </configuration>
    <dependencies>
        <!-- This dependency provides the implementation of compiler "jdt": -->
        <dependency>
            <groupId>org.eclipse.tycho</groupId>
            <artifactId>tycho-compiler-jdt</artifactId>
            <version>0.21.0</version>
        </dependency>
    </dependencies>
</plugin>
~~~

如果你使用Eclipse开发, m2e插件可能会提示上述的几行配置是非法的. 如果提示错误, 在`pom.xml`插入下面的几行.

~~~xml
<!-- 在pom.xml中的 "project/build/pluginManagement/plugins/plugin[groupId="org.eclipse.m2e", artifactId="lifecycle-mapping"]/configuration/lifecycleMappingMetadata/pluginExecutions"插入下面几行 -->

<pluginExecution>
    <pluginExecutionFilter>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <versionRange>[3.1,)</versionRange>
        <goals>
            <goal>testCompile</goal>
            <goal>compile</goal>
        </goals>
    </pluginExecutionFilter>
    <action>
        <ignore></ignore>
    </action>
</pluginExecution>
~~~

#### 在Eclipse IDE运行和调试Flink任务

首先确保你使用的是较新版本的 IDE(4.4.2或者更新的). 同时保证在Eclipse IDE中装载了Java 8(JRE) ，通过(`Window` -> `Preferences` -> `Java` -> `Installed JREs`)查看.

创建/导入Eclipse工程.

如果使用Maven,同样需要在`pom.xml`的`maven-compiler-plugin`中修改Java版本. 或者在项目上右击，选择`JRE System Library`，然后打开`Properties` 窗口，调整到Java 8(或者更新的版本)来支持Lambda表达式.

Eclipse JDT 编译器需要一个特殊的编译标识以便在class文件中存储类型信息. 使用编辑器打开`{project directoy}/.settings/org.eclipse.jdt.core.prefs`下的配置文件，然后添加下面几行:

~~~
org.eclipse.jdt.core.compiler.codegen.lambda.genericSignature=generate
~~~

同时修改Java 版本到`1.8` (或者更高):

~~~
org.eclipse.jdt.core.compiler.codegen.targetPlatform=1.8
org.eclipse.jdt.core.compiler.compliance=1.8
org.eclipse.jdt.core.compiler.source=1.8
~~~

保存文件, 在Eclipse IDE中刷新工程.

如果使用Maven, 右击你的工程选择`Maven` -> `Update Project...`.

如果上述配置正确, 下面的Flink程序可以正确执行:

~~~java
final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.fromElements(1, 2, 3).map((in) -> new Tuple1<String>(" " + in)).print();
env.execute();
~~~
