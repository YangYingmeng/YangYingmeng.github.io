---
layout: post
title: MyBatis
categories: [MyBatis源码]
description: 学习MyBatis源码
keywords: 源码, MyBatis
---

# 一、MyBatis源码之异常模块

### 1. IbatisException(已被废弃)

> 实现 RuntimeException 类，IBatis 的异常基类, 所在包: org.apache.ibatis.exceptions.IbatisException

```java
@Deprecated
public class IbatisException extends RuntimeException {

  private static final long serialVersionUID = 3880206998166270511L;

  public IbatisException() {
  }

  public IbatisException(String message) {
    super(message);
  }

  public IbatisException(String message, Throwable cause) {
    super(message, cause);
  }

  public IbatisException(Throwable cause) {
    super(cause);
  }

}
```



### 2. PersistenceException

> 继承 IbatisException 类，目前 MyBatis 真正的异常基类 所在包: org.apache.ibatis.exceptions.PersistenceException

```java
@SuppressWarnings("deprecation")
public class PersistenceException extends IbatisException {

  private static final long serialVersionUID = -7537395265357977271L;

  public PersistenceException() {
  }

  public PersistenceException(String message) {
    super(message);
  }

  public PersistenceException(String message, Throwable cause) {
    super(message, cause);
  }

  public PersistenceException(Throwable cause) {
    super(cause);
  }
}
```



### 3. ExceptionFactory

> 异常工厂, 所在包: org.apache.ibatis.exceptions.ExceptionFactory

```java
public class ExceptionFactory {

  private ExceptionFactory() {
    // Prevent Instantiation
  }

  /**
   * 包装异常成 PersistenceException
   *
   * @param message 消息
   * @param e 发生的异常
   * @return PersistenceException
   */
  public static RuntimeException wrapException(String message, Exception e) {
    return new PersistenceException(ErrorContext.instance().message(message).cause(e).toString(), e);
  }

}
```



### 4. TooManyResultsException

> 继承 PersistenceException 类，查询返回过多结果的异常。期望返回一条，实际返回了多条, 所在包: org.apache.ibatis.exceptions.TooManyResultsException

```java
public class TooManyResultsException extends PersistenceException {

  private static final long serialVersionUID = 8935197089745865786L;

  public TooManyResultsException() {
  }

  public TooManyResultsException(String message) {
    super(message);
  }

  public TooManyResultsException(String message, Throwable cause) {
    super(message, cause);
  }

  public TooManyResultsException(Throwable cause) {
    super(cause);
  }
}
```



### 5. ParsingException

> 继承 PersistenceException 类，解析异常, 所在包: org.apache.ibatis.parsing.ParsingException

```java
public class ParsingException extends PersistenceException {
  private static final long serialVersionUID = -176685891441325943L;

  public ParsingException() {
  }

  public ParsingException(String message) {
    super(message);
  }

  public ParsingException(String message, Throwable cause) {
    super(message, cause);
  }

  public ParsingException(Throwable cause) {
    super(cause);
  }
}
```



### 6. 其它

>其它异常基本都是基于PersistenceException
