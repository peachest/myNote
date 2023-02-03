## FROM

每个`FROM`指令声明了一个build阶段

`FROM`指定源镜像。构建时，如果源镜像不在本地，则docker会自动拉取源镜像。

一个dockerfile可以允许多个`FROM`指令，用于 **构建多个image**或者 **进行分阶段build**，每个`FROM`指令会清除前面指令产生的所有状态。



分阶段build时，后面的`FROM`会用到前面build出来的image，为了后续指令方便引用，可以**使用`AS`取别名**.

```dockerfile
FROM nginx:latest AS ng 
```



`FROM`指令前只允许出现`ARG`指令。且第一个`FROM`语句的`ARG`语句声明的值**无法在build阶段使用**，即可以在后续`FROM`，`ENTRYPOINT`，`CMD`中通过`${}`引用，但无法在`RUN`等build语句使用。

如果仍然需要在build阶段使用，则需要在`FROM`后面使用`ARG`语句声明同名变量（无需声明值）。



## RUN

每个`RUN`指令在源镜像上添加新的一层，因此最好将多个`RUN`指令合并，避免镜像层数过多。

`RUN`指令声明了build时执行的指令。

`RUN`有两种格式：

* shell：
* exec：

****

**shell格式**

`RUN`后为一条完整的shell命令，默认使用`/bin/sh -c`指令执行

可以使用`\`将单行指令分成多行

可以使用`ARG`中定义的变量



****

**exec格式**

```dockerfile
RUN ["可执行对象", "参数1", "参数2"]
```

exec格式可以避免破坏shell字符串。并且可以在**不包含任何shell**可执行文件的基础镜像上，执行`RUN`命令

使用JSON解析字符串数组，因此**必须使用双引号**，但解析后的所有字符都是字面量，即**无法使用`ARG`中定义的变量**



由于使用了JSON，**反斜杠`\`必须进行转义**。即`RUN ["c:\\path\\to\\executable.exe"]`





****





## CMD

`CMD`不属于build阶段。`CMD`在执行`docker run`时才发挥作用。

`CMD` **主要作用是为`ENTRYPOINT`提供默认参数**，也可以单独声明一条指令在docker run时执行。

`CMD`有三种格式：

* shell：同`RUN`指令的shell格式
* exec：同`RUN`指令的exec格式
* 提供默认参数：`CMD ["参数1", "参数2", ]`



**只有最后一条`CMD`才会执行**，其余`CMD`会被忽略

当`CMD`搭配`ENTRYPOINT`时，`docker run`中指定的参数会覆盖`CMD`提供的默认参数



## LABEL

`LABEL`指令添加image的metadata。如：版本号，描述，作者。使用键值对的形式

一条`LABEL`语句可以声明多个label

可以引用变量

存在空格时需要使用双引号

label会被继承，同名label中新声明的会覆盖旧的

`docker image inspect --format='' <image>`查看image中定义的所有label



## EXPOSE

`EXPOSE`指令声明了容器运行时监听的网络端口。

格式：`EXPOSE 80`或者 `EXPOSE 80/udp`

默认使用TCP协议

可以同时声明监听多个协议：

```dock
EXPOSE 80/tcp
EXPOSE 80/udp
```



使用`docker run -P`随机端口映射，则容器的80端口会暴露为tcp或者为udp

使用`docker run -p 80:80/tcp -p 80:80/udp`覆盖`EXPOSE`指令的声明



容器之间的通信，可以使用`docker network`建立通信网络，而不需要暴露端口。



## ENV

`ENV`指令使用键值对的形式声明环境变量，可以在**build阶段**或者**容器运行时**使用。

一条`ENV`指令可以声明多个环境变量

环境变量会被继承



## ADD

**每个`ADD`会添加新的一层**

`ADD`复制本地资源或者网络资源到镜像内部

基础格式：`ADD src1, src2, ..., dest `，即可以指定多个src路径，最后一个路径参数视为dest路径。

**所有src路径都应该是相对路径**，相对于`docker build`时传入的**上下文路径**，默认是dockerfile所在的目录路径。并且只能是当前上下文目录或者其子目录，而不能是上下文路径的上级目录，即不能使用`../*`，因为build的第一步是将上下文目录及其子目录发送到docker daemon中，不包含其上级目录。



本地的压缩文件或被自动解压后，但是远程URL的压缩资源不会被解压。压缩文件根据**文件内容**而非后缀名判断





dest可以是绝对路径，也可以是相对`WORKDIR/`的相对路径



根据官方 Dockerfile 最佳实践，除非真的需要**从远程 url 添加文件或自动提取压缩文件**才用 ADD，其他情况一律使用 COPY



所有`ADD`添加的目录或者文件的uid与gid都为0。使用`--chown`修改，`:`分割uid与gid

```dockerfile
ADD --chown=myuser:mygroup	# 使用username与groupname
ADD --chown=10:11 			# 使用uid与gid 
ADD --chown=10:mygroup 		# 混合

ADD --chown=10 				# gid与uid相同
```

使用的username与groupname必须在容器中存在，因为容器会使用其本身的`/etc/passwd`与`/etc/group`解析，如果文件中不存在对应的name，则ADD会失败



## COPY

**每个`COPY`会添加新的一层**

格式上与`ADD`相同

`COPY`只能复制本地文件

在多阶段build中，使用`COPY --from=<name>`，可以从前继的image中复制文件，而不是上下文目录。



## ENTRYPOINT

`ENTRYPOINT`有shell与exec两种格式

`ENTRYPOINT`声明了`docker run <image>`时容器执行的命令，且所有参数将传入并放置到`ENTRYPOINT`exec格式下的参数数组后面，并覆盖



规则：只需看`ENTRYPOINT`的格式

1. 没有声明`ENTRYPOINT`时，按照`CMD`执行
2. 使用shell格式声明`ENTRYPOINT`时，**无视`CMD`**
3. 使用exec格式声明`ENTRYPOINT`，`CMD`放在`ENTRYPOINT`的参数数组后面作为默认参数



## USER

`USER`设置user与group，使用名字或id或混合模式声明，`:`分割

`USER`设置的user与group可以被build阶段或runtime使用，包括：

* `RUN`指令
* `CMD`与`ENTRYPOINT`指令
* 容器运行时的默认user/group



windows中如果user不存在，必须先创建：

```dockerfile
RUN net user /add patrick	# 创建新用户
USER patrick				# 设置用户
```



## WORKDIR

`WORKDIR`设置dockerfile执行时使用的容器工作目录，例如`COPY`时dest为相对路径时，即相对于workdir

`WORKDIR`类似于cd，默认为容器的根目录`/`





## VOLUME

`VOLUME`声明容器中的数据卷，并且只能在启动容器时指定该数据卷在host上的挂载点，将数据卷中的数据持久化到host的挂载点上。挂载点无法在dockerfile中声明

host的挂载点必须满足下述其中一个条件：

* 目录不存在或空目录
* 非C盘的驱动器



`VOLUME /myvol`将持久化该语句之前所有对`/myvol`目录下的内容做的修改。**`VOLUME /myvol`语句之后任何对`/myvol`的修改将会被忽略。**





## SHELL

修改`RUN`，`CMD`，`ENTRYPOINT`在shell格式下使用的shell程序



windows下：

`SHELL ["powershell", "-command"]`

`SHELL ["cmd", "/S", "/C"]`





# 最佳实践













