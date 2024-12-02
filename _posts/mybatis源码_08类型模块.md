---
layout: post
title: MyBatis
categories: [MyBatis源码]
description: 学习MyBatis源码
keywords: 源码, MyBatis
---

# 一、MyBatis源码之类型模块

> 提供转换别名机制, 提供JDBC类型和Java类型的转换

### 1. TypeHandler

> 类型转换处理器, 所在包: org.apache.ibatis.type.TypeHandler

```java
public interface TypeHandler<T> {

  /**
   * 设置 PreparedStatement 的指定参数
   * Java Type => JDBC Type
   *
   * @param ps PreparedStatement 对象
   * @param i 参数占位符的位置
   * @param parameter 参数
   * @param jdbcType JDBC 类型
   * @throws SQLException 当发生 SQL 异常时
   */
  void setParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType) throws SQLException;

  /**
   * 获得 ResultSet 的指定字段的值
   * JDBC Type => Java Type
   *
   * @param rs ResultSet 对象
   * @param columnName 字段名
   * @return 值
   * @throws SQLException 当发生 SQL 异常时
   */
  T getResult(ResultSet rs, String columnName) throws SQLException;
  /**
   * 获得 ResultSet 的指定字段的值
   * JDBC Type => Java Type
   *
   * @param rs ResultSet 对象
   * @param columnIndex 字段位置
   * @return 值
   * @throws SQLException 当发生 SQL 异常时
   */
  T getResult(ResultSet rs, int columnIndex) throws SQLException;
  /**
   * 获得 CallableStatement 的指定字段的值
   * JDBC Type => Java Type
   *
   * @param cs CallableStatement 对象，支持调用存储过程
   * @param columnIndex 字段位置
   * @return 值
   * @throws SQLException
   */
  T getResult(CallableStatement cs, int columnIndex) throws SQLException;

}
```

### 2. BaseTypeHandler

> 实现 TypeHandler 接口，继承 TypeReference 抽象类，TypeHandler 基础抽象类, 所在包: org.apache.ibatis.type.BaseTypeHandler

- setParameter

  ```java
    @Override
    public void setParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType) throws SQLException {
      // 1. 参数为空时, 设置为null类型
      if (parameter == null) {
        if (jdbcType == null) {
          throw new TypeException("JDBC requires that the JdbcType must be specified for all nullable parameters.");
        }
        try {
          ps.setNull(i, jdbcType.TYPE_CODE);
        } catch (SQLException e) {
          throw new TypeException("Error setting null for parameter #" + i + " with JdbcType " + jdbcType + " . "
              + "Try setting a different JdbcType for this parameter or a different jdbcTypeForNull configuration property. "
              + "Cause: " + e, e);
        }
        // 参数非空时, 设置对应的参数
      } else {
        try {
          setNonNullParameter(ps, i, parameter, jdbcType);
        } catch (Exception e) {
          throw new TypeException("Error setting non null for parameter #" + i + " with JdbcType " + jdbcType + " . "
              + "Try setting a different JdbcType for this parameter or a different configuration property. " + "Cause: "
              + e, e);
        }
      }
    }
  
    /**
     * 由子类实现, 当发生异常时, 统一抛出 TypeException 异常
     */
    public abstract void setNonNullParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType)
        throws SQLException;
  ```

- getResult

  ```java
    @Override
    public T getResult(ResultSet rs, String columnName) throws SQLException {
      try {
        return getNullableResult(rs, columnName);
      } catch (Exception e) {
        throw new ResultMapException("Error attempting to get column '" + columnName + "' from result set.  Cause: " + e,
            e);
      }
    }
  
    @Override
    public T getResult(ResultSet rs, int columnIndex) throws SQLException {
      try {
        return getNullableResult(rs, columnIndex);
      } catch (Exception e) {
        throw new ResultMapException("Error attempting to get column #" + columnIndex + " from result set.  Cause: " + e,
            e);
      }
    }
  
    @Override
    public T getResult(CallableStatement cs, int columnIndex) throws SQLException {
      try {
        return getNullableResult(cs, columnIndex);
      } catch (Exception e) {
        throw new ResultMapException(
            "Error attempting to get column #" + columnIndex + " from callable statement.  Cause: " + e, e);
      }
    }
  
    /**
     * 由子类实现, 当发生异常时, 统一抛出 TypeException 异常
     */
    public abstract T getNullableResult(ResultSet rs, String columnName) throws SQLException;
    
    public abstract T getNullableResult(ResultSet rs, int columnIndex) throws SQLException;
  
    public abstract T getNullableResult(CallableStatement cs, int columnIndex) throws SQLException;
  ```



### 3. 子类

> TypeHandler 有非常多的子类，**当然所有子类都是继承自 BaseTypeHandler 抽象类**, 随便举几个例子

#### 3.1 IntegerTypeHandler

> 继承 BaseTypeHandler 抽象类，Integer 类型的 TypeHandler 实现类, 所在包: org.apache.ibatis.type.IntegerTypeHandler

```java
public class IntegerTypeHandler extends BaseTypeHandler<Integer> {

  @Override
  public void setNonNullParameter(PreparedStatement ps, int i, Integer parameter, JdbcType jdbcType)
      throws SQLException {
    // 直接设置参数
    ps.setInt(i, parameter);
  }

  @Override
  public Integer getNullableResult(ResultSet rs, String columnName) throws SQLException {
    // 获得字段的值
    int result = rs.getInt(columnName);
    // 先通过 rs 判断是否空，如果是空，则返回 null ，否则返回 result
    return result == 0 && rs.wasNull() ? null : result;
  }

  @Override
  public Integer getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
    int result = rs.getInt(columnIndex);
    return result == 0 && rs.wasNull() ? null : result;
  }

  @Override
  public Integer getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
    int result = cs.getInt(columnIndex);
    return result == 0 && cs.wasNull() ? null : result;
  }
}
```



#### 3.2 DateTypeHandler

> 继承 BaseTypeHandler 抽象类，Date 类型的 TypeHandler 实现类, 所在包: org.apache.ibatis.type.DateTypeHandler

```java
public class DateTypeHandler extends BaseTypeHandler<Date> {

  @Override
  public void setNonNullParameter(PreparedStatement ps, int i, Date parameter, JdbcType jdbcType) throws SQLException {
    // Date -> Timestamp, 设置到ps中
    ps.setTimestamp(i, new Timestamp(parameter.getTime()));
  }

  @Override
  public Date getNullableResult(ResultSet rs, String columnName) throws SQLException {
    // Timestamp -> Date
    return toDate(rs.getTimestamp(columnName));
  }

  @Override
  public Date getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
    // Timestamp -> Date
    return toDate(rs.getTimestamp(columnIndex));
  }

  @Override
  public Date getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
    // Timestamp -> Date
    return toDate(cs.getTimestamp(columnIndex));
  }

  private Date toDate(Timestamp timestamp) {
    return timestamp == null ? null : new Date(timestamp.getTime());
  }

}
```



#### 3.3 DateOnlyTypeHandler

> 继承 BaseTypeHandler 抽象类，Date 类型的 TypeHandler 实现类, 所在包: org.apache.ibatis.type.DateOnlyTypeHandler

```java
public class DateOnlyTypeHandler extends BaseTypeHandler<Date> {

  @Override
  public void setNonNullParameter(PreparedStatement ps, int i, Date parameter, JdbcType jdbcType) throws SQLException {
    // java Date -> sql Date
    ps.setDate(i, new java.sql.Date(parameter.getTime()));
  }

  @Override
  public Date getNullableResult(ResultSet rs, String columnName) throws SQLException {
    // sql Date -> java Date
    return toSqlDate(rs.getDate(columnName));
  }

  @Override
  public Date getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
    // sql Date -> java Date
    return toSqlDate(rs.getDate(columnIndex));
  }

  @Override
  public Date getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
    // sql Date -> java Date
    return toSqlDate(cs.getDate(columnIndex));
  }

  private java.sql.Date toSqlDate(Date date) {
    // sql Date -> java Date
    return date == null ? null : new java.sql.Date(date.getTime());
  }

}
```



#### 3.4 EnumTypeHandler

> 继承 BaseTypeHandler 抽象类，Enum 类型的 TypeHandler 实现类, 所在包: org.apache.ibatis.type.EnumTypeHandler

```java
public class EnumTypeHandler<E extends Enum<E>> extends BaseTypeHandler<E> {

  /**
   * 枚举类
   */
  private final Class<E> type;

  public EnumTypeHandler(Class<E> type) {
    if (type == null) {
      throw new IllegalArgumentException("Type argument cannot be null");
    }
    this.type = type;
  }

  @Override
  public void setNonNullParameter(PreparedStatement ps, int i, E parameter, JdbcType jdbcType) throws SQLException {
    //  Enum -> String, 如果没有指定jdbcType则用枚举名称替代, 如果指定增加枚举类型使用setObject()方法
    if (jdbcType == null) {
      ps.setString(i, parameter.name());
    } else {
      ps.setObject(i, parameter.name(), jdbcType.TYPE_CODE); // see r3589
    }
  }

  @Override
  public E getNullableResult(ResultSet rs, String columnName) throws SQLException {
    // 获得 String 值
    String s = rs.getString(columnName);
    // String值 转为 Enum 
    return s == null ? null : Enum.valueOf(type, s);
  }

  @Override
  public E getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
    // 获得 String 值
    String s = rs.getString(columnIndex);
    // String值 转为 Enum
    return s == null ? null : Enum.valueOf(type, s);
  }

  @Override
  public E getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
    // 获得 String 值
    String s = cs.getString(columnIndex);
    // String值 转为 Enum
    return s == null ? null : Enum.valueOf(type, s);
  }
}
```



#### 3.5 EnumOrdinalTypeHandler

> 继承 BaseTypeHandler 抽象类，Enum 类型的 TypeHandler 实现类, 所在包: org.apache.ibatis.type.EnumOrdinalTypeHandler

```java
public class EnumOrdinalTypeHandler<E extends Enum<E>> extends BaseTypeHandler<E> {

  /**
   * 枚举类
   */
  private final Class<E> type;
  /**
   * {@link #type} 下所有的枚举
   *
   * @see Class#getEnumConstants()
   */
  private final E[] enums;

  public EnumOrdinalTypeHandler(Class<E> type) {
    if (type == null) {
      throw new IllegalArgumentException("Type argument cannot be null");
    }
    this.type = type;
    this.enums = type.getEnumConstants();
    if (this.enums == null) {
      throw new IllegalArgumentException(type.getSimpleName() + " does not represent an enum type.");
    }
  }

  @Override
  public void setNonNullParameter(PreparedStatement ps, int i, E parameter, JdbcType jdbcType) throws SQLException {
    // 将 Enum 转换成 int 类型
    ps.setInt(i, parameter.ordinal());
  }

  @Override
  public E getNullableResult(ResultSet rs, String columnName) throws SQLException {
    // 获得 int 的值
    int ordinal = rs.getInt(columnName);
    // 将 int 准换成 Enum 类型
    if (ordinal == 0 && rs.wasNull()) {
      return null;
    }
    return toOrdinalEnum(ordinal);
  }

  @Override
  public E getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
    // 获得 int 的值
    int ordinal = rs.getInt(columnIndex);
    // 将 int 转换成 Enum 类型
    if (ordinal == 0 && rs.wasNull()) {
      return null;
    }
    return toOrdinalEnum(ordinal);
  }

  @Override
  public E getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
    // 获得 int 的值
    int ordinal = cs.getInt(columnIndex);
    // 将 int 转换成 Enum 类型
    if (ordinal == 0 && cs.wasNull()) {
      return null;
    }
    return toOrdinalEnum(ordinal);
  }

  /**
   * 返回对应的枚举
   */
  private E toOrdinalEnum(int ordinal) {
    try {
      return enums[ordinal];
    } catch (Exception ex) {
      throw new IllegalArgumentException(
          "Cannot convert " + ordinal + " to " + type.getSimpleName() + " by ordinal value.", ex);
    }
  }
}
```



### 4. TypeReference

> 引用泛型抽象类。目的很简单，就是解析类上定义的泛型, 所在包: org.apache.ibatis.type.TypeReference

```java
public abstract class TypeReference<T> {

  /**
   * 所有类型的通用超接口, 可以通过反射获取和操作类型信息
   * 例如A<T> extend TypeReference<T>
   * 在编译阶段, 字节码中只有A, <T>被擦除,
   * 但是在A加载时, 由于有继承会明确<T>的类型, 并存放在类元数据中(JVM内存)
   * 关键点在于继承, 继承时子类会明确其泛型类型
   */
  private final Type rawType;

  protected TypeReference() {
    rawType = getSuperclassTypeParameter(getClass());
  }

  Type getSuperclassTypeParameter(Class<?> clazz) {
    // 通过反射从父类中获取泛型的类型参数 <T>
    Type genericSuperclass = clazz.getGenericSuperclass();
    if (genericSuperclass instanceof Class) {
      // try to climb up the hierarchy until meet something useful
      // 如果父类不是 TypeReference 类型, 会一直查找到第一个带有泛型类型的类, 并将T返回
      if (TypeReference.class != genericSuperclass) {
        return getSuperclassTypeParameter(clazz.getSuperclass());
      }

      throw new TypeException("'" + getClass() + "' extends TypeReference but misses the type parameter. "
          + "Remove the extension or add a type parameter to it.");
    }
    // 获取 T
    Type rawType = ((ParameterizedType) genericSuperclass).getActualTypeArguments()[0];
    // TODO remove this when Reflector is fixed to return Types
    // 必须是泛型, 才获取 T
    if (rawType instanceof ParameterizedType) {
      rawType = ((ParameterizedType) rawType).getRawType();
    }

    return rawType;
  }

  public final Type getRawType() {
    return rawType;
  }

  @Override
  public String toString() {
    return rawType.toString();
  }

}
```



### 5. 注解

> 分析type包中的三个注解



#### 5.1 @MappedTypes

> 匹配的 Java Type 类型的注解, 所在包: org.apache.ibatis.type.@MappedTypes

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE) // 注册到类
public @interface MappedTypes {
  /**
   * Returns java types to map {@link TypeHandler}.
   *
   * @return java types 匹配的 Java Type 类型的数组
   */
  Class<?>[] value();
}
```

#### 5.2 @MappedJdbcTypes

> 匹配的 JDBC Type 类型的注解, 所在包: org.apache.ibatis.type.@MappedJdbcTypes

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface MappedJdbcTypes {
  /**
   * Returns jdbc types to map {@link TypeHandler}.
   *
   * @return jdbc types 匹配的 JDBC Type 类型的注解
   */
  JdbcType[] value();

  /**
   * Returns whether map to jdbc null type.
   * 是否包含 {@link java.sql.JDBCType#NULL}
   * @return {@code true} if map, {@code false} if otherwise
   */
  boolean includeNullJdbcType() default false;
}
```

#### 5.3 Alias

> 别名的注解, 所在包: org.apache.ibatis.type.@Alias

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Alias {
  /**
   * Return the alias name.
   *
   * @return the alias name
   */
  String value();
}
```



### 6. JdbcType

> Jdbc Type 枚举, 所在包: org.apache.ibatis.type.JdbcType

```java
public enum JdbcType {
  /*
   * This is added to enable basic support for the ARRAY data type - but a custom type handler is still required
   */
  ARRAY(Types.ARRAY),

  BIT(Types.BIT),

  TINYINT(Types.TINYINT),

  SMALLINT(Types.SMALLINT),

  INTEGER(Types.INTEGER),

  BIGINT(Types.BIGINT),

  FLOAT(Types.FLOAT),

  REAL(Types.REAL),

  DOUBLE(Types.DOUBLE),

  NUMERIC(Types.NUMERIC),

  DECIMAL(Types.DECIMAL),

  CHAR(Types.CHAR),

  VARCHAR(Types.VARCHAR),

  LONGVARCHAR(Types.LONGVARCHAR),

  DATE(Types.DATE),

  TIME(Types.TIME),

  TIMESTAMP(Types.TIMESTAMP),

  BINARY(Types.BINARY),

  VARBINARY(Types.VARBINARY),

  LONGVARBINARY(Types.LONGVARBINARY),

  NULL(Types.NULL),

  OTHER(Types.OTHER),

  BLOB(Types.BLOB),

  CLOB(Types.CLOB),

  BOOLEAN(Types.BOOLEAN),

  CURSOR(-10), // Oracle

  UNDEFINED(Integer.MIN_VALUE + 1000),

  NVARCHAR(Types.NVARCHAR), // JDK6

  NCHAR(Types.NCHAR), // JDK6

  NCLOB(Types.NCLOB), // JDK6

  STRUCT(Types.STRUCT),

  JAVA_OBJECT(Types.JAVA_OBJECT),

  DISTINCT(Types.DISTINCT),

  REF(Types.REF),

  DATALINK(Types.DATALINK),

  ROWID(Types.ROWID), // JDK6

  LONGNVARCHAR(Types.LONGNVARCHAR), // JDK6

  SQLXML(Types.SQLXML), // JDK6

  DATETIMEOFFSET(-155), // SQL Server 2008

  TIME_WITH_TIMEZONE(Types.TIME_WITH_TIMEZONE), // JDBC 4.2 JDK8

  TIMESTAMP_WITH_TIMEZONE(Types.TIMESTAMP_WITH_TIMEZONE); // JDBC 4.2 JDK8

  /**
   * 类型编号
   */
  public final int TYPE_CODE;
  /**
   * 代码编号和 {@link JdbcType} 的映射
   */
  private static final Map<Integer, JdbcType> codeLookup = new HashMap<>();

  static {
    // 初始化 codeLookup
    for (JdbcType type : JdbcType.values()) {
      codeLookup.put(type.TYPE_CODE, type);
    }
  }

  JdbcType(int code) {
    this.TYPE_CODE = code;
  }

  public static JdbcType forCode(int code) {
    return codeLookup.get(code);
  }

}
```



### 7. TypeHandlerRegistry

> TypeHandler 注册表，相当于管理 TypeHandler 的容器，从其中能获取到对应的 TypeHandler , 所在包: org.apache.ibatis.type.TypeHandlerRegistry

- 构造方法

  ```java
    /**
     * JDBC Type 和 {@link TypeHandler} 的映射
     * {@link #register(JdbcType, TypeHandler)}
     */
    private final Map<JdbcType, TypeHandler<?>> jdbcTypeHandlerMap = new EnumMap<>(JdbcType.class);
    /**
     * {@link TypeHandler} 的映射
     * KEY1：JDBC Type
     * KEY2：Java Type
     * VALUE：{@link TypeHandler} 对象
     */
    private final Map<Type, Map<JdbcType, TypeHandler<?>>> typeHandlerMap = new ConcurrentHashMap<>();
    /**
     * {@link UnknownTypeHandler} 对象
     */
    private final TypeHandler<Object> unknownTypeHandler;
    /**
     * 所有 TypeHandler 的“集合”
     * KEY：{@link TypeHandler#getResult()}
     * VALUE：{@link TypeHandler} 对象
     */
    private final Map<Class<?>, TypeHandler<?>> allTypeHandlersMap = new HashMap<>();
    /**
     * 空 TypeHandler 集合的标识，即使 {@link #typeHandlerMap} 中，某个 KEY1 对应的 Map<JdbcType, TypeHandler<?>> 为空。
     * @see #getJdbcHandlerMap(Type)
     */
    private static final Map<JdbcType, TypeHandler<?>> NULL_TYPE_HANDLER_MAP = Collections.emptyMap();
    /**
     * 默认的枚举类型的 TypeHandler 对象
     */
    private Class<? extends TypeHandler> defaultEnumTypeHandler = EnumTypeHandler.class;
  
    public TypeHandlerRegistry(Configuration configuration) {
      this.unknownTypeHandler = new UnknownTypeHandler(configuration);
  
      register(Boolean.class, new BooleanTypeHandler());
      register(boolean.class, new BooleanTypeHandler());
      register(JdbcType.BOOLEAN, new BooleanTypeHandler());
      register(JdbcType.BIT, new BooleanTypeHandler());
  
      register(Byte.class, new ByteTypeHandler());
      register(byte.class, new ByteTypeHandler());
      register(JdbcType.TINYINT, new ByteTypeHandler());
  
      register(Short.class, new ShortTypeHandler());
      register(short.class, new ShortTypeHandler());
      register(JdbcType.SMALLINT, new ShortTypeHandler());
  
      register(Integer.class, new IntegerTypeHandler());
      register(int.class, new IntegerTypeHandler());
      register(JdbcType.INTEGER, new IntegerTypeHandler());
  
      register(Long.class, new LongTypeHandler());
      register(long.class, new LongTypeHandler());
  
      register(Float.class, new FloatTypeHandler());
      register(float.class, new FloatTypeHandler());
      register(JdbcType.FLOAT, new FloatTypeHandler());
  
      register(Double.class, new DoubleTypeHandler());
      register(double.class, new DoubleTypeHandler());
      register(JdbcType.DOUBLE, new DoubleTypeHandler());
  
      register(Reader.class, new ClobReaderTypeHandler());
      register(String.class, new StringTypeHandler());
      register(String.class, JdbcType.CHAR, new StringTypeHandler());
      register(String.class, JdbcType.CLOB, new ClobTypeHandler());
      register(String.class, JdbcType.VARCHAR, new StringTypeHandler());
      register(String.class, JdbcType.LONGVARCHAR, new StringTypeHandler());
      register(String.class, JdbcType.NVARCHAR, new NStringTypeHandler());
      register(String.class, JdbcType.NCHAR, new NStringTypeHandler());
      register(String.class, JdbcType.NCLOB, new NClobTypeHandler());
      register(JdbcType.CHAR, new StringTypeHandler());
      register(JdbcType.VARCHAR, new StringTypeHandler());
      register(JdbcType.CLOB, new ClobTypeHandler());
      register(JdbcType.LONGVARCHAR, new StringTypeHandler());
      register(JdbcType.NVARCHAR, new NStringTypeHandler());
      register(JdbcType.NCHAR, new NStringTypeHandler());
      register(JdbcType.NCLOB, new NClobTypeHandler());
  
      register(Object.class, JdbcType.ARRAY, new ArrayTypeHandler());
      register(JdbcType.ARRAY, new ArrayTypeHandler());
  
      register(BigInteger.class, new BigIntegerTypeHandler());
      register(JdbcType.BIGINT, new LongTypeHandler());
  
      register(BigDecimal.class, new BigDecimalTypeHandler());
      register(JdbcType.REAL, new BigDecimalTypeHandler());
      register(JdbcType.DECIMAL, new BigDecimalTypeHandler());
      register(JdbcType.NUMERIC, new BigDecimalTypeHandler());
  
      register(InputStream.class, new BlobInputStreamTypeHandler());
      register(Byte[].class, new ByteObjectArrayTypeHandler());
      register(Byte[].class, JdbcType.BLOB, new BlobByteObjectArrayTypeHandler());
      register(Byte[].class, JdbcType.LONGVARBINARY, new BlobByteObjectArrayTypeHandler());
      register(byte[].class, new ByteArrayTypeHandler());
      register(byte[].class, JdbcType.BLOB, new BlobTypeHandler());
      register(byte[].class, JdbcType.LONGVARBINARY, new BlobTypeHandler());
      register(JdbcType.LONGVARBINARY, new BlobTypeHandler());
      register(JdbcType.BLOB, new BlobTypeHandler());
  
      register(Object.class, unknownTypeHandler);
      register(Object.class, JdbcType.OTHER, unknownTypeHandler);
      register(JdbcType.OTHER, unknownTypeHandler);
  
      register(Date.class, new DateTypeHandler());
      register(Date.class, JdbcType.DATE, new DateOnlyTypeHandler());
      register(Date.class, JdbcType.TIME, new TimeOnlyTypeHandler());
      register(JdbcType.TIMESTAMP, new DateTypeHandler());
      register(JdbcType.DATE, new DateOnlyTypeHandler());
      register(JdbcType.TIME, new TimeOnlyTypeHandler());
  
      register(java.sql.Date.class, new SqlDateTypeHandler());
      register(java.sql.Time.class, new SqlTimeTypeHandler());
      register(java.sql.Timestamp.class, new SqlTimestampTypeHandler());
  
      register(String.class, JdbcType.SQLXML, new SqlxmlTypeHandler());
  
      register(Instant.class, new InstantTypeHandler());
      register(LocalDateTime.class, new LocalDateTimeTypeHandler());
      register(LocalDate.class, new LocalDateTypeHandler());
      register(LocalTime.class, new LocalTimeTypeHandler());
      register(OffsetDateTime.class, new OffsetDateTimeTypeHandler());
      register(OffsetTime.class, new OffsetTimeTypeHandler());
      register(ZonedDateTime.class, new ZonedDateTimeTypeHandler());
      register(Month.class, new MonthTypeHandler());
      register(Year.class, new YearTypeHandler());
      register(YearMonth.class, new YearMonthTypeHandler());
      register(JapaneseDate.class, new JapaneseDateTypeHandler());
  
      // issue #273
      register(Character.class, new CharacterTypeHandler());
      register(char.class, new CharacterTypeHandler());
    }
  
  ```

- getInstance

  ```java
    public <T> TypeHandler<T> getInstance(Class<?> javaTypeClass, Class<?> typeHandlerClass) {
      if (javaTypeClass != null) {
        try {
          // 获得class类型的构造方法
          Constructor<?> c = typeHandlerClass.getConstructor(Class.class);
          return (TypeHandler<T>) c.newInstance(javaTypeClass);
        } catch (NoSuchMethodException ignored) {
          // ignored
        } catch (Exception e) {
          throw new TypeException("Failed invoking constructor for handler " + typeHandlerClass, e);
        }
      }
      // 获得空参的构造方法
      try {
        Constructor<?> c = typeHandlerClass.getConstructor();
        return (TypeHandler<T>) c.newInstance();
      } catch (Exception e) {
        throw new TypeException("Unable to find a usable constructor for " + typeHandlerClass, e);
      }
    }
  ```

- register

  > 注册 TypeHandler 。TypeHandlerRegistry 中有大量该方法的重载实现

  ```java
    private void register(Type javaType, JdbcType jdbcType, TypeHandler<?> handler) {
      // 添加 handler 到 typeHandlerMap
      if (javaType != null) {
        Map<JdbcType, TypeHandler<?>> map = typeHandlerMap.get(javaType);
        // 如果不存在, 则进行创建
        if (map == null || map == NULL_TYPE_HANDLER_MAP) {
          map = new HashMap<>();
        }
        // 添加到 typeHandlerMap
        map.put(jdbcType, handler);
        typeHandlerMap.put(javaType, map);
      }
      // 添加到 allTypeHandlersMap
      allTypeHandlersMap.put(handler.getClass(), handler);
    }
  ```



### 8. TypeAliasRegistry

> 类型与别名的注册表。通过别名，我们在 Mapper XML 中的 `resultType` 和 `parameterType` 属性，直接使用，而不用写全类名, 所在包: org.apache.ibatis.type.TypeAliasRegistry

- 构造方法

  ```java
  /**
     * 类型与别名的映射
     */
    private final Map<String, Class<?>> typeAliases = new HashMap<>();
  
    /**
     * 初始化默认的类型与别名
     * 另外，在 {@link org.apache.ibatis.session.Configuration} 构造方法中，也有默认的注册
     */
    public TypeAliasRegistry() {
      // 可以用String 替代 java.lang.String, 不用写这些类的类全名
      registerAlias("string", String.class);
  
      registerAlias("byte", Byte.class);
      registerAlias("char", Character.class);
      registerAlias("character", Character.class);
      registerAlias("long", Long.class);
      registerAlias("short", Short.class);
      registerAlias("int", Integer.class);
      registerAlias("integer", Integer.class);
      registerAlias("double", Double.class);
      registerAlias("float", Float.class);
      registerAlias("boolean", Boolean.class);
  
      registerAlias("byte[]", Byte[].class);
      registerAlias("char[]", Character[].class);
      registerAlias("character[]", Character[].class);
      registerAlias("long[]", Long[].class);
      registerAlias("short[]", Short[].class);
      registerAlias("int[]", Integer[].class);
      registerAlias("integer[]", Integer[].class);
      registerAlias("double[]", Double[].class);
      registerAlias("float[]", Float[].class);
      registerAlias("boolean[]", Boolean[].class);
  
      registerAlias("_byte", byte.class);
      registerAlias("_char", char.class);
      registerAlias("_character", char.class);
      registerAlias("_long", long.class);
      registerAlias("_short", short.class);
      registerAlias("_int", int.class);
      registerAlias("_integer", int.class);
      registerAlias("_double", double.class);
      registerAlias("_float", float.class);
      registerAlias("_boolean", boolean.class);
  
      registerAlias("_byte[]", byte[].class);
      registerAlias("_char[]", char[].class);
      registerAlias("_character[]", char[].class);
      registerAlias("_long[]", long[].class);
      registerAlias("_short[]", short[].class);
      registerAlias("_int[]", int[].class);
      registerAlias("_integer[]", int[].class);
      registerAlias("_double[]", double[].class);
      registerAlias("_float[]", float[].class);
      registerAlias("_boolean[]", boolean[].class);
  
      registerAlias("date", Date.class);
      registerAlias("decimal", BigDecimal.class);
      registerAlias("bigdecimal", BigDecimal.class);
      registerAlias("biginteger", BigInteger.class);
      registerAlias("object", Object.class);
  
      registerAlias("date[]", Date[].class);
      registerAlias("decimal[]", BigDecimal[].class);
      registerAlias("bigdecimal[]", BigDecimal[].class);
      registerAlias("biginteger[]", BigInteger[].class);
      registerAlias("object[]", Object[].class);
  
      registerAlias("map", Map.class);
      registerAlias("hashmap", HashMap.class);
      registerAlias("list", List.class);
      registerAlias("arraylist", ArrayList.class);
      registerAlias("collection", Collection.class);
      registerAlias("iterator", Iterator.class);
  
      registerAlias("ResultSet", ResultSet.class);
    }
  ```

- registerAlias

  ```java
    public void registerAlias(Class<?> type) {
      // 默认为简单类名
      String alias = type.getSimpleName();
      // 如果有注解, 使用注册上的名字
      Alias aliasAnnotation = type.getAnnotation(Alias.class);
      if (aliasAnnotation != null) {
        alias = aliasAnnotation.value();
      }
      // 注册类型与别名的注册表
      registerAlias(alias, type);
    }
  
    public void registerAlias(String alias, Class<?> value) {
      if (alias == null) {
        throw new TypeException("The parameter alias cannot be null");
      }
      // issue #748
      // 转换成小写
      String key = alias.toLowerCase(Locale.ENGLISH);
      if (typeAliases.containsKey(key) && typeAliases.get(key) != null && !typeAliases.get(key).equals(value)) {
        throw new TypeException(
            "The alias '" + alias + "' is already mapped to the value '" + typeAliases.get(key).getName() + "'.");
      }
      typeAliases.put(key, value);
    }
  
    public void registerAlias(String alias, String value) {
      try {
        // 通过类名的字符串, 获得对应的类
        registerAlias(alias, Resources.classForName(value));
      } catch (ClassNotFoundException e) {
        throw new TypeException("Error registering type alias " + alias + " for " + value + ". Cause: " + e, e);
      }
    }
  ```

- registerAliases

  ```java
    /**
     * 扫描指定包下的所有类, 并进行注册(别名与类的映射)
     */
    public void registerAliases(String packageName) {
      registerAliases(packageName, Object.class);
    }
  
    /**
     * 注册指定包下的别名与类的映射。另外，要求类必须是 {@param superType} 类型（包括子类）
     */
    public void registerAliases(String packageName, Class<?> superType) {
      // 获得指定包下的类名
      ResolverUtil<Class<?>> resolverUtil = new ResolverUtil<>();
      resolverUtil.find(new ResolverUtil.IsA(superType), packageName);
      Set<Class<? extends Class<?>>> typeSet = resolverUtil.getClasses();
      // 遍历, 逐个注册类型与别名的注册表
      for (Class<?> type : typeSet) {
        // Ignore inner classes and interfaces (including package-info.java)
        // Skip also inner classes. See issue #6
        // 排除匿名类 接口 内部类
        if (!type.isAnonymousClass() && !type.isInterface() && !type.isMemberClass()) {
          registerAlias(type);
        }
      }
    }
  ```

- resolveAlias

  ```java
    @SuppressWarnings("unchecked")
    // throws class cast exception as well if types cannot be assigned
    public <T> Class<T> resolveAlias(String string) {
      try {
        if (string == null) {
          return null;
        }
        // issue #748
        // 转换成小写
        String key = string.toLowerCase(Locale.ENGLISH);
        Class<T> value;
        // 先从typeAliases获取
        if (typeAliases.containsKey(key)) {
          value = (Class<T>) typeAliases.get(key);
          // 直接获得对应类
        } else {
          value = (Class<T>) Resources.classForName(string);
        }
        return value;
      } catch (ClassNotFoundException e) {
        throw new TypeException("Could not resolve type alias '" + string + "'.  Cause: " + e, e);
      }
    }
  ```

  

### 9. SimpleTypeRegistry

> 简单类型注册表, 所在包: org.apache.ibatis.type.SimpleTypeRegistry

```java
public class SimpleTypeRegistry {

  /**
   * 简单类型的集合
   */
  private static final Set<Class<?>> SIMPLE_TYPE_SET = new HashSet<>();

  // 初始化常用类到 SIMPLE_TYPE_SET 中
  static {
    SIMPLE_TYPE_SET.add(String.class);
    SIMPLE_TYPE_SET.add(Byte.class);
    SIMPLE_TYPE_SET.add(Short.class);
    SIMPLE_TYPE_SET.add(Character.class);
    SIMPLE_TYPE_SET.add(Integer.class);
    SIMPLE_TYPE_SET.add(Long.class);
    SIMPLE_TYPE_SET.add(Float.class);
    SIMPLE_TYPE_SET.add(Double.class);
    SIMPLE_TYPE_SET.add(Boolean.class);
    SIMPLE_TYPE_SET.add(Date.class);
    SIMPLE_TYPE_SET.add(Class.class);
    SIMPLE_TYPE_SET.add(BigInteger.class);
    SIMPLE_TYPE_SET.add(BigDecimal.class);
  }

  private SimpleTypeRegistry() {
    // Prevent Instantiation
  }

  /*
   * Tells us if the class passed in is a known common type
   * @param clazz The class to check
   * @return True if the class is known
   */
  public static boolean isSimpleType(Class<?> clazz) {
    return SIMPLE_TYPE_SET.contains(clazz);
  }

}
```



### 10. ByteArrayUtils

> Byte 数组的工具类, 所在包: org.apache.ibatis.type.ByteArrayUtils

```java
class ByteArrayUtils {

  private ByteArrayUtils() {
    // Prevent Instantiation
  }

  static byte[] convertToPrimitiveArray(Byte[] objects) {
    final byte[] bytes = new byte[objects.length];
    for (int i = 0; i < objects.length; i++) {
      bytes[i] = objects[i];
    }
    return bytes;
  }

  static Byte[] convertToObjectArray(byte[] bytes) {
    final Byte[] objects = new Byte[bytes.length];
    for (int i = 0; i < bytes.length; i++) {
      objects[i] = bytes[i];
    }
    return objects;
  }
}
```

