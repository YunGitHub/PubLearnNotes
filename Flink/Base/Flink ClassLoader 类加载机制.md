在运行 Flink 应用程序时，JVM 会随着时间的推移加载各种不同的类。可以根据它们的来源分为三种：
- Java Classpath：这是 Java 的通用类路径，包括 JDK 类库，以及 Flink 的 lib 目录下的所有代码（Apache Flink 的类以及一些依赖）。
- Flink Plugin 组件：Flink 的 plugins 目录下的插件代码。Flink 的插件机制会在启动时动态加载一次。
- 动态用户代码：这些都是包含在动态提交作业 JAR 中的类（通过 REST、CLI、Web UI 提交）。它们在每个作业中动态加载（和卸载）。

一般来说，每当我们启动 Flink 进程并提交作业时，作业的类都是动态加载的。如果 Flink 进程与作业/应用程序一起启动，或者如果应用程序产生 Flink 组件（JobManager、TaskManager 等），那么所有作业的类都在 Java Classpath 中。插件组件中的代码由每个插件的专用类加载器动态加载一次。

以下是有关不同部署模式的更多详细信息：

## 反转类加载和类加载器解析顺序

在涉及动态类加载的设置中（插件组件、会话设置中的 Flink 作业），通常有两个类加载器：
- Java 的应用程序类加载器：加载类路径 classpath 下的所有类
- Flink 的动态类加载器(动态插件/用户代码)：用于从插件或用户代码 jar 中加载类。动态类加载器把应用程序类加载器作为父类加载器。

默认情况下，Flink 会颠倒类加载顺序，即首先查看动态类加载器，如果类不是动态加载代码的一部分时才会委托给父类加载器，即应用程序类加载器。

反向类加载的好处是插件和作业可以使用与 Flink 核心库不同的版本，这在不同版本的库不兼容时非常有用。该机制有助于避免常见的依赖冲突错误，如 IllegalAccessError 或 NoSuchMethodError。代码的不同部分只是拥有单独的类副本（Flink 的核心或其依赖项之一可以使用与用户代码或插件代码不同的副本）。在大多数情况下，这很有效，不需要用户进行额外的配置。

但是，在某些情况下，反向类加载会导致问题（见下文，“X 不能强制转换为 X”）。对于动态类加载器，我们可以通过 Flink config 中的 classloader.resolve-order 将 ClassLoader 解析顺序配置为 parent-first（Flink 默认为 child-first）恢复到 Java 的默认模式。

需要注意的是，某些类必须以父类优先的方式解析（首先通过父类加载器），因为它们在 Flink 核心库与插件/用户代码或面向插件/用户代码的 API 之间共享。这些类的包需要通过 classloader.parent-first-patterns-default 和 classloader.parent-first-patterns-additional 来配置。要添加新的包实现 parent-first 加载，需要设置 classloader.parent-first-patterns-additional 配置选项。


在具有动态类加载的设置中，我们可能会看到 com.foo.X 无法转换为 com.foo.X 的异常。这意味着 com.foo.X 类的多个版本已由不同的类加载器加载，并且该类的类型试图相互分配。

一个常见的原因是库与 Flink 的反向类加载方法不兼容。您可以关闭反向类加载来验证这一点（在 Flink 配置中设置 classloader.resolve-order: parent-first）或从反向类加载中排除库（在 Flink 配置中设置 classloader.parent-first-patterns-additional）。

另一个原因可能是缓存的对象实例，由一些库（如 Apache Avro）或实习对象（例如通过 Guava 的实习生）产生。这里的解决方案是要么设置一个没有任何动态类加载的设置，要么确保相应的库完全是动态加载代码的一部分。后者意味着该库不能添加到 Flink 的 lib 文件夹中，而必须是应用程序 fat-jar/uber-jar 的一部分

## 避免用户代码的动态类加载

所有组件(JobManger、TaskManager、Client、ApplicationMaster 等)在启动时都会记录其类路径 classpath 设置。可以在日志开头的环境信息中找到。当运行 JobManager 和 TaskManagers 专用于一些特定作业的设置时，可以将用户代码的 JAR 直接放入 lib 目录下，以确保它们是类路径的一部分而不是动态加载。

通常可以将作业的 JAR 文件放入 lib 目录下。JAR 自然成为应用程序类加载器（AppClassLoader）和动态类加载器（FlinkUserCodeClassLoader）的一部分。因为应用程序类加载器是动态加载器的父类加载器，按照双亲委派模型类的加载会委派到应用程序类加载器上，所以 lib 目录下的类只会被应用程序类加载器加载一次。对于那些无法放入 lib 目录下的作业 JAR（例如，因为设置是由多个作业使用的会话），仍然可以将常用库放入 lib 目录下，并避免动态加载这些类。

## 在用户代码中手动加载类

在某些情况下，转换函数、Source 或 Sink 需要手动加载类（通过反射动态加载）。为了实现这个目标，需要有一个访问作业类的类加载器。在这种情况下，可以将函数（或者 Source、Sink）设为 RichFunction（例如 RichMapFunction 或 RichWindowFunction）并通过 getRuntimeContext().getUserCodeClassLoader() 访问用户代码类加载器。

## X cannot be cast to X exceptions

在具有动态类加载的设置中，您可能会看到样式 com.foo.X 无法转换为 com.foo.X 的异常。这意味着 com.foo.X 类的多个版本已由不同的类加载器加载，并且该类的类型试图相互分配。

一个常见的原因是库与 Flink 的反向类加载方法不兼容。您可以关闭反向类加载来验证这一点（在 Flink 配置中设置 classloader.resolve-order: parent-first）或从反向类加载中排除库（在 Flink 配置中设置 classloader.parent-first-patterns-additional）。

另一个原因可能是缓存的对象实例，由一些库（如 Apache Avro）或实习对象（例如通过 Guava 的实习生）产生。这里的解决方案是要么设置一个没有任何动态类加载的设置，要么确保相应的库完全是动态加载代码的一部分。后者意味着该库不能添加到 Flink 的 /lib 文件夹中，而必须是应用程序 fat-jar/uber-jar 的一部分




Flink Client respects Classloading Policy

> [FLINK-13749](https://issues.apache.org/jira/browse/FLINK-13749)

The Flink client now also respects the configured classloading policy, i.e., parent-first or child-first classloading. Previously, only cluster components such as the job manager or task manager supported this setting. This does mean that users might get different behaviour in their programs, in which case they should configure the classloading policy explicitly to use parent-first classloading, which was the previous (hard-coded) behaviour.


https://nightlies.apache.org/flink/flink-docs-release-1.13/docs/ops/debugging/debugging_classloading/
https://blog.csdn.net/weixin_34331744/article/details/112069015
https://blog.csdn.net/CarloPan/article/details/117295628
https://zhuanlan.zhihu.com/p/477362410
https://blog.csdn.net/chenxyz707/article/details/109043868
。。。
