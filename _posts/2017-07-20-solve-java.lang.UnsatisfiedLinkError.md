---
layout: post
title: 解决java.lang.UnsatisfiedLinkError org.apache.hadoop.io.nativeio.NativeIO$Windows.access0(Ljava/lang/String;I)Z
categories: HBase
description: java.lang.UnsatisfiedLinkError org.apache.hadoop.io.nativeio.NativeIO$Windows.access0(Ljava/lang/String;I)Z
keywords: HBase,MapReduce,Java
---

## 解决java.lang.UnsatisfiedLinkError: org.apache.hadoop.io.nativeio.NativeIO$Windows.access0(Ljava/lang/String;I)Z
最近需要学HBase，打算在windows下开发，用linux测试以减少bug。代码很简单：
```java
public class TestMiniCluster {
	private static final byte[] TableName = "test-table".getBytes();
	private static final byte[] FamilyName = "F".getBytes();

	private static HBaseTestingUtility hBaseTestingUtility;
	private static HTable hTable;
	private static MiniHBaseCluster cluster;
	private static FileSystem fs;
	private static Connection connection;
	private static Admin admin;

	@BeforeClass
	public static void setUpClass() throws Exception{
		hBaseTestingUtility = new HBaseTestingUtility();
		cluster = hBaseTestingUtility.startMiniCluster();
		hTable = hBaseTestingUtility.createTable(TableName, FamilyName);
		fs = FileSystem.get(hBaseTestingUtility.getConfiguration());
		connection = ConnectionFactory.createConnection(cluster.getConfiguration());
		admin = connection.getAdmin();
	}

	@Test
	public void testCluster() throws Exception{
		System.out.println("hello");
	}
}
```
大意是启动一个小型集群，然后报了这么个错：
>java.lang.UnsatisfiedLinkError: org.apache.hadoop.io.nativeio.NativeIO$Windows.access0(Ljava/lang/String;I)Z
>	at org.apache.hadoop.io.nativeio.NativeIO$Windows.access0(Native Method)
>	at org.apache.hadoop.io.nativeio.NativeIO$Windows.access(NativeIO.java:570)
>	at org.apache.hadoop.fs.FileUtil.canWrite(FileUtil.java:996)
>   ...

看起来是链接错误。我的机器是windows10系统，上网查了一下，最后用[这篇文章](http://www.raincent.com/content-85-8162-3.html)的解决方案搞定的，总结一下：

1. 下载winutils.exe等文件的windows编译版，需要和本地的hadoop版本相同，我用的是[hadoop.dll-and-winutils.exe-for-hadoop2.7.3-on-windows_X64](https://github.com/rucyang/hadoop.dll-and-winutils.exe-for-hadoop2.7.3-on-windows_X64)。

2. 把这些文件放到%HADOOP_HOME%\bin目录下，其中%HADOOP_HOME%是hadoop所在的目录，并且需要配置在环境变量中（不需要把%HADOOP_HOME%\bin放到path里）。

3. 注意到报错信息的第三行是NativeIO.java:570，所以下载这个包的源代码（如果用Intellij IDEA开发的话就很方便），找到这个文件放到项目的源代码里，并把第570行  
   ```return access0(path, desiredAccess.accessRight());```  
   改成  
   ```return true;```

这样应该就可以了。