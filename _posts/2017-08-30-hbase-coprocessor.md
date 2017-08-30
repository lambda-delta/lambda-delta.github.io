---
layout: post
title: HBase 协处理器（Coprocessor）学习
categories: HBase
description: HBase 协处理器（Coprocessor）学习。
keywords: HBase, Coprocessor
---

## HBase Coprocessor学习

本文主要参考《HBase权威指南》。

因工作原因，需要学习Coprocessor。简单理解，Coprocessor是运行在物理存储端的代码，毕竟直接运行在存储端有很多好处（至少访问速度快了）。

那么什么时候会运行Coprocessor呢？

* 当表发生一些操作时触发coprocessor，这类又称为观察者模式（observer），类似于sql里的触发器（trigger）
* 远端调用coprocessor，这类又称为终端模式（endpoint），类似于sql的存储过程（stored procedure）

工作中主要用到第二种情况，所以前者先放一放，以后再说。

### 1. Coprocessor接口

所有协处理器类必须要实现这个接口，源代码如下：

```java
public interface Coprocessor {
  static final int VERSION = 1;

  /** Highest installation priority */
  static final int PRIORITY_HIGHEST = 0;
  /** High (system) installation priority */
  static final int PRIORITY_SYSTEM = Integer.MAX_VALUE / 4;
  /** Default installation priority for user coprocessors */
  static final int PRIORITY_USER = Integer.MAX_VALUE / 2;
  /** Lowest installation priority */
  static final int PRIORITY_LOWEST = Integer.MAX_VALUE;

  /**
   * Lifecycle state of a given coprocessor instance.
   */
  public enum State {
    UNINSTALLED,
    INSTALLED,
    STARTING,
    ACTIVE,
    STOPPING,
    STOPPED
  }

  // Interface
  void start(CoprocessorEnvironment env) throws IOException;

  void stop(CoprocessorEnvironment env) throws IOException;
}
```

常量包括一些优先级和状态。start和stop在协处理器开始和结束时调用，其中涉及到CoprocessorEnvironment类，其源代码为：
```java
public interface CoprocessorEnvironment {

  /** @return the Coprocessor interface version */
  public int getVersion();

  /** @return the HBase version as a string (e.g. "0.21.0") */
  public String getHBaseVersion();

  /** @return the loaded coprocessor instance */
  public Coprocessor getInstance();

  /** @return the priority assigned to the loaded coprocessor */
  public int getPriority();

  /** @return the load sequence number */
  public int getLoadSequence();

  /** @return the configuration */
  public Configuration getConfiguration();

  /**
   * @return an interface for accessing the given table
   * @throws IOException
   */
  public HTableInterface getTable(byte[] tableName) throws IOException;
}
```
简单理解可以认为是协处理器当前环境的一些接口

### 2. CoprocessorProtocal接口

其本身是个空的接口：
```java
public interface CoprocessorProtocol extends VersionedProtocol {
  public static final long VERSION = 1L;
}
```

该接口可以定义需要暴露给用户的方法，客户端可以通过HTable（确切的说，是HTableInterface接口）调用获得，相关源代码如下：
```java
public interface HTableInterface extends Closeable {
...
  /**
   * Creates and returns a proxy to the CoprocessorProtocol instance running in the
   * region containing the specified row.  The row given does not actually have
   * to exist.  Whichever region would contain the row based on start and end keys will
   * be used.  Note that the {@code row} parameter is also not passed to the
   * coprocessor handler registered for this protocol, unless the {@code row}
   * is separately passed as an argument in a proxy method call.  The parameter
   * here is just used to locate the region used to handle the call.
   *
   * @param protocol The class or interface defining the remote protocol
   * @param row The row key used to identify the remote region location
   * @return A CoprocessorProtocol instance
   */
  <T extends CoprocessorProtocol> T coprocessorProxy(Class<T> protocol, byte[] row);

    /**
   * Invoke the passed
   * {@link org.apache.hadoop.hbase.client.coprocessor.Batch.Call} against
   * the {@link CoprocessorProtocol} instances running in the selected regions.
   * All regions beginning with the region containing the <code>startKey</code>
   * row, through to the region containing the <code>endKey</code> row (inclusive)
   * will be used.  If <code>startKey</code> or <code>endKey</code> is
   * <code>null</code>, the first and last regions in the table, respectively,
   * will be used in the range selection.
   *
   * @param protocol the CoprocessorProtocol implementation to call
   * @param startKey start region selection with region containing this row
   * @param endKey select regions up to and including the region containing
   * this row
   * @param callable wraps the CoprocessorProtocol implementation method calls
   * made per-region
   * @param <T> CoprocessorProtocol subclass for the remote invocation
   * @param <R> Return type for the
   * {@link org.apache.hadoop.hbase.client.coprocessor.Batch.Call#call(Object)}
   * method
   * @return a <code>Map</code> of region names to
   * {@link org.apache.hadoop.hbase.client.coprocessor.Batch.Call#call(Object)} return values
   */
  <T extends CoprocessorProtocol, R> Map<byte[],R> coprocessorExec(
      Class<T> protocol, byte[] startKey, byte[] endKey, Batch.Call<T,R> callable)
      throws IOException, Throwable;

  /**
   * Invoke the passed
   * {@link org.apache.hadoop.hbase.client.coprocessor.Batch.Call} against
   * the {@link CoprocessorProtocol} instances running in the selected regions.
   * All regions beginning with the region containing the <code>startKey</code>
   * row, through to the region containing the <code>endKey</code> row
   * (inclusive)
   * will be used.  If <code>startKey</code> or <code>endKey</code> is
   * <code>null</code>, the first and last regions in the table, respectively,
   * will be used in the range selection.
   *
   * <p>
   * For each result, the given
   * {@link org.apache.hadoop.hbase.client.coprocessor.Batch.Callback#update(byte[], byte[], Object)}
   * method will be called.
   *</p>
   *
   * @param protocol the CoprocessorProtocol implementation to call
   * @param startKey start region selection with region containing this row
   * @param endKey select regions up to and including the region containing
   * this row
   * @param callable wraps the CoprocessorProtocol implementation method calls
   * made per-region
   * @param callback an instance upon which
   * {@link org.apache.hadoop.hbase.client.coprocessor.Batch.Callback#update(byte[], byte[], Object)} with the
   * {@link org.apache.hadoop.hbase.client.coprocessor.Batch.Call#call(Object)}
   * return value for each region
   * @param <T> CoprocessorProtocol subclass for the remote invocation
   * @param <R> Return type for the
   * {@link org.apache.hadoop.hbase.client.coprocessor.Batch.Call#call(Object)}
   * method
   */
  <T extends CoprocessorProtocol, R> void coprocessorExec(
      Class<T> protocol, byte[] startKey, byte[] endKey,
      Batch.Call<T,R> callable, Batch.Callback<R> callback)
      throws IOException, Throwable;

...
}
```
Coprocessor所在的region是通过行健决定的，coprocessorProxy和coprocessorExec的区别在于前者使用单行健，而后者使用一段行健作为参数。值得注意的是row可以不存在。

### 3. BaseEndpointCoprocessor

在实现Endpoint Coprocessor时，可以继承这个虚基类，这个类提供了一些默认接口的实现方式。

```java
public abstract class BaseEndpointCoprocessor implements Coprocessor,
    CoprocessorProtocol, VersionedProtocol {
  /**
   * This Interfaces' version. Version changes when the Interface changes.
   */
  // All HBase Interfaces used derive from HBaseRPCProtocolVersion.  It
  // maintained a single global version number on all HBase Interfaces.  This
  // meant all HBase RPC was broke though only one of the three RPC Interfaces
  // had changed.  This has since been undone.
  public static final long VERSION = 28L;

  private CoprocessorEnvironment env;

  /**
   * @return env Coprocessor environment.
   */
  public CoprocessorEnvironment getEnvironment() {
    return env;
  }

  @Override
  public void start(CoprocessorEnvironment env) {
    this.env = env;
  }

  @Override
  public void stop(CoprocessorEnvironment env) { }

  @Override
  public ProtocolSignature getProtocolSignature(
      String protocol, long version, int clientMethodsHashCode)
  throws IOException {
    return new ProtocolSignature(VERSION, null);
  }

  @Override
  public long getProtocolVersion(String protocol, long clientVersion)
  throws IOException {
    return VERSION;
  }
}
```

### 4. Endpoint Coprocessor实现方式

用户自定义Endpoint Coprocessor时，需要实现两个类/接口：

1. 继承自CoprocessorProtocal接口，假定为XXXProtocal，这个接口需要声明自己要用到的方法

2. 继承自BaseEndpointCoprocessor和XXXProtocal的类，假定为XXXProtocalImpl，这个类需要实现XXXProtocal里声明的方法

### 5. Coprocessor加载

#### 5.1 从配置文件加载

只能在HBase启动时生效。修改hbase-site.xml，添加类似这样的属性：
```xml
<property>
   <name>hbase.coprocessor.user.region.classes</name>
   <value>org.apache.hadoop.hbase.coprocessor.AggregateImplementation</value>
 </property>
```

#### 5.2 通过代码加载

只在创建table时生效。在创建table时，可以选择加载的协处理器，代码类似：
```java
...
HTableDescriptor tableDesc = new HTableDescriptor(objectTableName);
tableDesc.addFamily(...);
tableDesc.addCoprocessor("org.apache.hadoop.hbase.coprocessor.AggregateImplementation");
hBaseAdmin.createTable(tableDesc);
...
```

#### 5.3 通过HBase Shell在线加载

相关细节比较麻烦，这里只说一下大致流程：

1. 编译协处理器相关代码，打包成jar
2. 在hbase所运行的hdfs上，建立一个目录，专门存放协处理器，并修改相应的权限
3. 把第一步中编译好的jar放到第二步中的目录里，注意相关权限
4. 进入hbase shell
5. 关闭要操作的表```disable 'xxx_table'```
6. 添加协处理器：```alter 'xxx_table', METHOD => 'table_att', 'Coprocessor'=>'hdfs:///your_path/xxx.jar|your.xxxProtocolImpl|1073741823'```，其中table_att为固定值不能改，jar包所在的hdfs路径根据实际情况自己改，类名要写全，最后一个数字是协处理器的优先级，随意写
7. 开启要操作的表```enable 'xxx_table'```
8. 查看设置是否成功：```describe 'xxx_table'```，如果输出结果正常，说明加载成功