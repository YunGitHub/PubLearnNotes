
## 1. 现象

在启动 Hive 运行 SQL 时抛出如下异常：
```java
Exception in thread "main" java.lang.RuntimeException: org.apache.hadoop.hive.ql.metadata.HiveException: java.lang.RuntimeException: Unable to instantiate org.apache.hadoop.hive.ql.metadata.SessionHiveMetaStoreClient
	at org.apache.hadoop.hive.ql.session.SessionState.setupAuth(SessionState.java:885)
	at org.apache.hadoop.hive.ql.session.SessionState.getAuthorizationMode(SessionState.java:1617)
	at org.apache.hadoop.hive.ql.session.SessionState.isAuthorizationModeV2(SessionState.java:1628)
	at org.apache.hadoop.hive.ql.processors.CommandUtil.authorizeCommand(CommandUtil.java:55)
	at org.apache.hadoop.hive.ql.processors.AddResourceProcessor.run(AddResourceProcessor.java:67)
	at org.apache.hadoop.hive.cli.CliDriver.processLocalCmd(CliDriver.java:288)
	at org.apache.hadoop.hive.cli.CliDriver.processCmd(CliDriver.java:184)
	at org.apache.hadoop.hive.cli.CliDriver.processLine(CliDriver.java:403)
	at org.apache.hadoop.hive.cli.CliDriver.executeDriver(CliDriver.java:821)
	at org.apache.hadoop.hive.cli.CliDriver.run(CliDriver.java:759)
	at org.apache.hadoop.hive.cli.CliDriver.main(CliDriver.java:686)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.apache.hadoop.util.RunJar.run(RunJar.java:226)
	at org.apache.hadoop.util.RunJar.main(RunJar.java:141)
Caused by: org.apache.hadoop.hive.ql.metadata.HiveException: java.lang.RuntimeException: Unable to instantiate org.apache.hadoop.hive.ql.metadata.SessionHiveMetaStoreClient
	at org.apache.hadoop.hive.ql.session.SessionState.setAuthorizerV2Config(SessionState.java:917)
	at org.apache.hadoop.hive.ql.session.SessionState.setupAuth(SessionState.java:877)
	... 16 more
Caused by: org.apache.hadoop.hive.ql.metadata.HiveException: java.lang.RuntimeException: Unable to instantiate org.apache.hadoop.hive.ql.metadata.SessionHiveMetaStoreClient
	at org.apache.hadoop.hive.ql.metadata.Hive.registerAllFunctionsOnce(Hive.java:236)
	at org.apache.hadoop.hive.ql.metadata.Hive.<init>(Hive.java:388)
	at org.apache.hadoop.hive.ql.metadata.Hive.create(Hive.java:332)
	at org.apache.hadoop.hive.ql.metadata.Hive.getInternal(Hive.java:312)
	at org.apache.hadoop.hive.ql.metadata.Hive.get(Hive.java:288)
	at org.apache.hadoop.hive.ql.session.SessionState.setAuthorizerV2Config(SessionState.java:913)
	... 17 more
Caused by: java.lang.RuntimeException: Unable to instantiate org.apache.hadoop.hive.ql.metadata.SessionHiveMetaStoreClient
	at org.apache.hadoop.hive.metastore.MetaStoreUtils.newInstance(MetaStoreUtils.java:1708)
	at org.apache.hadoop.hive.metastore.RetryingMetaStoreClient.<init>(RetryingMetaStoreClient.java:83)
	at org.apache.hadoop.hive.metastore.RetryingMetaStoreClient.getProxy(RetryingMetaStoreClient.java:133)
	at org.apache.hadoop.hive.metastore.RetryingMetaStoreClient.getProxy(RetryingMetaStoreClient.java:104)
	at org.apache.hadoop.hive.ql.metadata.Hive.createMetaStoreClient(Hive.java:3600)
	at org.apache.hadoop.hive.ql.metadata.Hive.getMSC(Hive.java:3652)
	at org.apache.hadoop.hive.ql.metadata.Hive.getMSC(Hive.java:3632)
	at org.apache.hadoop.hive.ql.metadata.Hive.getAllFunctions(Hive.java:3894)
	at org.apache.hadoop.hive.ql.metadata.Hive.reloadFunctions(Hive.java:248)
	at org.apache.hadoop.hive.ql.metadata.Hive.registerAllFunctionsOnce(Hive.java:231)
	... 22 more
Caused by: java.lang.reflect.InvocationTargetException
	at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
	at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62)
	at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
	at java.lang.reflect.Constructor.newInstance(Constructor.java:423)
	at org.apache.hadoop.hive.metastore.MetaStoreUtils.newInstance(MetaStoreUtils.java:1706)
	... 31 more
Caused by: MetaException(message:Could not connect to meta store using any of the URIs provided. Most recent failure: org.apache.thrift.transport.TTransportException: java.net.ConnectException: Connection refused (Connection refused)
	at org.apache.thrift.transport.TSocket.open(TSocket.java:226)
	at org.apache.hadoop.hive.metastore.HiveMetaStoreClient.open(HiveMetaStoreClient.java:480)
	at org.apache.hadoop.hive.metastore.HiveMetaStoreClient.<init>(HiveMetaStoreClient.java:247)
	at org.apache.hadoop.hive.ql.metadata.SessionHiveMetaStoreClient.<init>(SessionHiveMetaStoreClient.java:70)
	at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
	at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62)
	at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
	at java.lang.reflect.Constructor.newInstance(Constructor.java:423)
	at org.apache.hadoop.hive.metastore.MetaStoreUtils.newInstance(MetaStoreUtils.java:1706)
	at org.apache.hadoop.hive.metastore.RetryingMetaStoreClient.<init>(RetryingMetaStoreClient.java:83)
	at org.apache.hadoop.hive.metastore.RetryingMetaStoreClient.getProxy(RetryingMetaStoreClient.java:133)
	at org.apache.hadoop.hive.metastore.RetryingMetaStoreClient.getProxy(RetryingMetaStoreClient.java:104)
	at org.apache.hadoop.hive.ql.metadata.Hive.createMetaStoreClient(Hive.java:3600)
	at org.apache.hadoop.hive.ql.metadata.Hive.getMSC(Hive.java:3652)
	at org.apache.hadoop.hive.ql.metadata.Hive.getMSC(Hive.java:3632)
	at org.apache.hadoop.hive.ql.metadata.Hive.getAllFunctions(Hive.java:3894)
	at org.apache.hadoop.hive.ql.metadata.Hive.reloadFunctions(Hive.java:248)
	at org.apache.hadoop.hive.ql.metadata.Hive.registerAllFunctionsOnce(Hive.java:231)
	at org.apache.hadoop.hive.ql.metadata.Hive.<init>(Hive.java:388)
	at org.apache.hadoop.hive.ql.metadata.Hive.create(Hive.java:332)
	at org.apache.hadoop.hive.ql.metadata.Hive.getInternal(Hive.java:312)
	at org.apache.hadoop.hive.ql.metadata.Hive.get(Hive.java:288)
	at org.apache.hadoop.hive.ql.session.SessionState.setAuthorizerV2Config(SessionState.java:913)
	at org.apache.hadoop.hive.ql.session.SessionState.setupAuth(SessionState.java:877)
	at org.apache.hadoop.hive.ql.session.SessionState.getAuthorizationMode(SessionState.java:1617)
	at org.apache.hadoop.hive.ql.session.SessionState.isAuthorizationModeV2(SessionState.java:1628)
	at org.apache.hadoop.hive.ql.processors.CommandUtil.authorizeCommand(CommandUtil.java:55)
	at org.apache.hadoop.hive.ql.processors.AddResourceProcessor.run(AddResourceProcessor.java:67)
	at org.apache.hadoop.hive.cli.CliDriver.processLocalCmd(CliDriver.java:288)
	at org.apache.hadoop.hive.cli.CliDriver.processCmd(CliDriver.java:184)
	at org.apache.hadoop.hive.cli.CliDriver.processLine(CliDriver.java:403)
	at org.apache.hadoop.hive.cli.CliDriver.executeDriver(CliDriver.java:821)
	at org.apache.hadoop.hive.cli.CliDriver.run(CliDriver.java:759)
	at org.apache.hadoop.hive.cli.CliDriver.main(CliDriver.java:686)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.apache.hadoop.util.RunJar.run(RunJar.java:226)
	at org.apache.hadoop.util.RunJar.main(RunJar.java:141)
Caused by: java.net.ConnectException: Connection refused (Connection refused)
	at java.net.PlainSocketImpl.socketConnect(Native Method)
	at java.net.AbstractPlainSocketImpl.doConnect(AbstractPlainSocketImpl.java:350)
	at java.net.AbstractPlainSocketImpl.connectToAddress(AbstractPlainSocketImpl.java:206)
	at java.net.AbstractPlainSocketImpl.connect(AbstractPlainSocketImpl.java:188)
	at java.net.SocksSocketImpl.connect(SocksSocketImpl.java:392)
	at java.net.Socket.connect(Socket.java:589)
	at org.apache.thrift.transport.TSocket.open(TSocket.java:221)
	... 39 more
)
	at org.apache.hadoop.hive.metastore.HiveMetaStoreClient.open(HiveMetaStoreClient.java:529)
	at org.apache.hadoop.hive.metastore.HiveMetaStoreClient.<init>(HiveMetaStoreClient.java:247)
	at org.apache.hadoop.hive.ql.metadata.SessionHiveMetaStoreClient.<init>(SessionHiveMetaStoreClient.java:70)
	... 36 more
```

## 2. 分析

从上面的异常信息 Could not connect to meta store using any of the URIs provided，我们可以知道 Hive 连接 MetaStore 失败，所以需要判断一下 Metadata 服务是否在运行。使用如下命令查看 MetaStore 服务是否在运行：
```
localhost:script wy$ ps -ef | grep HiveMetaStore
  501  4796   947   0  2:09下午 ttys001    0:00.00 grep HiveMetaStore
```
确实发现 MetaStore 服务没有在运行。

## 3. 解决方案

需要用户自己分析 MetaStore 服务是因为异常退出还是因为没有启动。在这我们是没有启动，所以需要使用如下启动 MetaStore 服务：
```
hive --service metastore -p 9083 &
```
> 有关 MetaStore 服务的详细信息，可以查阅[Hive 元数据服务 MetaStore](https://smartsi.blog.csdn.net/article/details/124440004)
