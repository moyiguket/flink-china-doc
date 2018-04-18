---
title:  "在Windows上运行Flink"
nav-parent_id: start
nav-pos: 12
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

如果你需要在windows机器上以本地模式运行Flink，你需要[下载](http://flink.apache.org/downloads.html) 并解压Flink二进制包。 完成上述步骤后，你可以使用**Windows批处理文件**(`.bat`)，或者使用**Cygwin** 来启动Flink Jobmanager。

## 使用Windows批处理文件启动

想要通过*Windows批处理程序*以本地模式启动Flink，打开控制台，进入Flink下的`bin/` 目录，然后运行`start-local.bat`。

注意：Java运行时的``bin``目录必须在Windows的``%PATH%``环境变量中。 根据这个[向导](http://www.java.com/en/download/help/path.xml) 将JAVA添加到 ``%PATH%``环境变量中。

~~~bash
$ cd flink
$ cd bin
$ start-local.bat
Starting Flink job manager. Web interface by default on http://localhost:8081/.
Do not close this batch window. Stop job manager by pressing Ctrl+C.
~~~

做完上述步骤，你需要重新打开一个cmd窗口，执行`flink.bat`来运行jobs。

{% top %}

## 通过Cygwin和Unix脚本启动

使用*Cygwin*你需要先开启Cygwin窗口，进入到Flink目录，然后执行`start-local.sh`脚本：

~~~bash
$ cd flink
$ bin/start-local.sh
Starting jobmanager.
~~~

{% top %}

## 通过git源代码安装

如果通过git仓库安装Flink，并且使用的是Windows git 命令, Cygwin可能会报如下错误：

~~~bash
c:/flink/bin/start-local.sh: line 30: $'\r': command not found
~~~

这个错误的原因是，在Windows环境下git自动将UNIX的行终止符转为Windows格式的行终止符。 问题是Cygwin只能处理UNIX格式的行终止符。解决方案是通过以下三个步骤来调整Cygwin的配置以便处理错误的行终止符:

1. 启动Cygwin窗口。

2. 通过以下命令获得用户目录

    ~~~bash
    cd; pwd
    ~~~

    执行完毕后将返回Cygwin根目录下的一个路径。

3. 使用NotePad，WordPad或者其他的文本编辑器打开用户主目录下的`.bash_profile`文件，然后添加下面的内容: (如果文件不存在，你需要自己去创建它)

~~~bash
export SHELLOPTS
set -o igncr
~~~

保存文件然后打开一个新窗口。

{% top %}
