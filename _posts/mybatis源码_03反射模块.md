---
layout: post
title: MyBatis
categories: [MyBatis源码]
description: 学习MyBatis源码
keywords: 源码, MyBatis
---

# 一、MyBatis源码之反射模块

> 封装了Java原生的反射, 并对其进行优化



### 1. Reflector

> 基础支持层, 包含整个MyBatis的基础模块, 为核心处理层提供良好的支持

```java
public class Reflector {

  private static final MethodHandle isRecordMethodHandle = getIsRecordMethodHandle();
  /**
   * 对应的类
   */
  private final Class<?> type;
  /**
   * 可读属性集合
   */
  private final String[] readablePropertyNames;
  /**
   * 可写属性集合
   */
  private final String[] writablePropertyNames;
  /**
   * 属性对应setting方法的映射, key为属性名称, value为Invoke对象
   */
  private final Map<String, Invoker> setMethods = new HashMap<>();
  /**
   * 属性对应getting方法的映射, key为属性名称, value为Invoke对象
   */
  private final Map<String, Invoker> getMethods = new HashMap<>();
  /**
   * 属性对应的 setting 方法的方法参数类型的映射。{@link #setMethods}
   * key为属性名称, value方法参数类型
   */
  private final Map<String, Class<?>> setTypes = new HashMap<>();
  /**
   * 属性对应的 getting 方法的返回值类型的映射。{@link #getMethods}
   * key为属性名称, value为返回值的类型
   */
  private final Map<String, Class<?>> getTypes = new HashMap<>();
  /**
   * 默认构造方法
   */
  private Constructor<?> defaultConstructor;
  /**
   * 不区分大小写的属性集合
   */
  private final Map<String, String> caseInsensitivePropertyMap = new HashMap<>();

  public Reflector(Class<?> clazz) {
    // 设置对应的类
    type = clazz;
    // 1. 初始化 defaultConstructor
    addDefaultConstructor(clazz);
    Method[] classMethods = getClassMethods(clazz);
    if (isRecord(type)) {
      addRecordGetMethods(classMethods);
    } else {
      // 2. 初始化getMethods 和 getTypes, 通过遍历 getting 方法
      addGetMethods(classMethods);
      // 3. 初始化setMethods 和 setTypes, 通过遍历 setting 方法
      addSetMethods(classMethods);
      // 4. 初始化getMethods + getTypes 和 setMethods + setTypes ，通过遍历 fields 属性。
      addFields(clazz);
    }
    // 5. 初始化 readablePropertyNames、writeablePropertyNames、caseInsensitivePropertyMap 属性
    readablePropertyNames = getMethods.keySet().toArray(new String[0]);
    writablePropertyNames = setMethods.keySet().toArray(new String[0]);
    for (String propName : readablePropertyNames) {
      caseInsensitivePropertyMap.put(propName.toUpperCase(Locale.ENGLISH), propName);
    }
    for (String propName : writablePropertyNames) {
      caseInsensitivePropertyMap.put(propName.toUpperCase(Locale.ENGLISH), propName);
    }
  }
  // ............
}
```



#### 1.1 addRecordGetMethods

> #addDefaultConstructor(Class<?> clazz) 方法，查找默认无参构造方法

```java
  private void addDefaultConstructor(Class<?> clazz) {
    // 获取所有构造方法
    Constructor<?>[] constructors = clazz.getDeclaredConstructors();
    // 将无参构造方法复制给默认构造方法
    Arrays.stream(constructors).filter(constructor -> constructor.getParameterTypes().length == 0).findAny()
        .ifPresent(constructor -> this.defaultConstructor = constructor);
  }
```



#### 1.2 addGetMethods

> #addGetMethods(Class<?> cls) 方法，初始化 getMethods 和 getTypes ，通过遍历 getting 方法

```java
private void addGetMethods(Method[] methods) {
    Map<String, List<Method>> conflictingGetters = new HashMap<>();
    // get方法没有参数 && 以 get和 is方法名开头
    Arrays.stream(methods).filter(m -> m.getParameterTypes().length == 0 && PropertyNamer.isGetter(m.getName()))
        .forEach(m -> addMethodConflict(conflictingGetters, PropertyNamer.methodToProperty(m.getName()), m));
    // 解决get方法冲突
    resolveGetterConflicts(conflictingGetters);
  }

  /**
   * 将get方法添加到数组中
   */
  private void addMethodConflict(Map<String, List<Method>> conflictingMethods, String name, Method method) {
    if (isValidPropertyName(name)) {
      List<Method> list = MapUtil.computeIfAbsent(conflictingMethods, name, k -> new ArrayList<>());
      list.add(method);
    }
  }
```



#### 1.3 getClassMethods

> #getClassMethods(Class<?> cls) 方法，获得所有方法

```java
  private Method[] getClassMethods(Class<?> clazz) {
    // key: 方法签名 value: 方法
    Map<String, Method> uniqueMethods = new HashMap<>();
    // 循环类及其父类, 知道Object为止
    Class<?> currentClass = clazz;
    while (currentClass != null && currentClass != Object.class) {
      // 1. 记录当前类定义的方法
      addUniqueMethods(uniqueMethods, currentClass.getDeclaredMethods());

      // we also need to look for interface methods -
      // because the class may be abstract
      // 2. 记录当前接口定义的方法
      Class<?>[] interfaces = currentClass.getInterfaces();
      for (Class<?> anInterface : interfaces) {
        addUniqueMethods(uniqueMethods, anInterface.getMethods());
      }
      // 获得父类
      currentClass = currentClass.getSuperclass();
    }
    // 转为Method数组返回
    Collection<Method> methods = uniqueMethods.values();

    return methods.toArray(new Method[0]);
  }


  // 将方法添加到uniqueMethods中
  private void addUniqueMethods(Map<String, Method> uniqueMethods, Method[] methods) {
    for (Method currentMethod : methods) {
      // https://www.zhihu.com/question/54895701/answer/141623158
      if (!currentMethod.isBridge()) {
        // 获得方法签名
        String signature = getSignature(currentMethod);
        // check to see if the method is already known
        // if it is known, then an extended class must have
        // overridden a method
        if (!uniqueMethods.containsKey(signature)) {
          uniqueMethods.put(signature, currentMethod);
        }
      }
    }
  }

  /**
   * 获得方法签名格式如下
   * returnType#方法名:参数名1,参数名2,参数名3
   * void#checkPackageAccess:java.lang.ClassLoader,boolean
   */
  private String getSignature(Method method) {
    StringBuilder sb = new StringBuilder();
    Class<?> returnType = method.getReturnType();
    sb.append(returnType.getName()).append('#');
    sb.append(method.getName());
    Class<?>[] parameters = method.getParameterTypes();
    for (int i = 0; i < parameters.length; i++) {
      sb.append(i == 0 ? ':' : ',').append(parameters[i].getName());
    }
    return sb.toString();
  }
```



#### 1.4 resolveGetterConflicts

> #resolveGetterConflicts(Map<String, List<Method>>) 方法，解决 getting 冲突方法。最终，一个属性，只保留一个对应的方法

```java
private void resolveGetterConflicts(Map<String, List<Method>> conflictingGetters) {
    // 遍历每个属性，查找其最匹配的方法。因为子类可以覆写父类的方法，所以一个属性，可能对应多个 getting 方法
    for (Entry<String, List<Method>> entry : conflictingGetters.entrySet()) {
      // 最匹配的方法
      Method winner = null;
      String propName = entry.getKey();
      boolean isAmbiguous = false;
      for (Method candidate : entry.getValue()) {
        if (winner == null) {
          // 如果winner为空 说明candidate最为合适
          winner = candidate;
          continue;
        }
        // 基于返回类型的比较
        Class<?> winnerType = winner.getReturnType();
        Class<?> candidateType = candidate.getReturnType();
        // 如果类型相同
        if (candidateType.equals(winnerType)) {
          // 返回值了诶选哪个相同，应该在 getClassMethods 方法中，已经合并。所以抛出 ReflectionException 异常
          if (!boolean.class.equals(candidateType)) {
            isAmbiguous = true;
            break;
          }
          if (candidate.getName().startsWith("is")) {
            winner = candidate;
          }
          // 选择子类, 子类继承会放大父类方法的返回值
        } else if (candidateType.isAssignableFrom(winnerType)) {
          // OK getter type is descendant
        } else if (winnerType.isAssignableFrom(candidateType)) {
          winner = candidate;
        } else {
          isAmbiguous = true;
          break;
        }
      }
      // 添加到 getMethods 和 getTypes中
      addGetMethod(propName, winner, isAmbiguous);
    }
  }

  private void addGetMethod(String name, Method method, boolean isAmbiguous) {
    MethodInvoker invoker = isAmbiguous ? new AmbiguousMethodInvoker(method, MessageFormat.format(
        "Illegal overloaded getter method with ambiguous type for property ''{0}'' in class ''{1}''. This breaks the JavaBeans specification and can cause unpredictable results.",
        name, method.getDeclaringClass().getName())) : new MethodInvoker(method);
    getMethods.put(name, invoker);
    Type returnType = TypeParameterResolver.resolveReturnType(method, type);
    getTypes.put(name, typeToClass(returnType));
  }
```



#### 1.5 addSetMethods

> #addSetMethods(Class<?> cls) 方法，初始化 `setMethods` 和 `setTypes` ，通过遍历 setting 方法

```java
private void addSetMethods(Method[] methods) {
    // key: 属性   value: setting方法
    Map<String, List<Method>> conflictingSetters = new HashMap<>();
    // 筛选 setting方法, 并将setting方法添加到conflictingSetters中
    Arrays.stream(methods).filter(m -> m.getParameterTypes().length == 1 && PropertyNamer.isSetter(m.getName()))
        .forEach(m -> addMethodConflict(conflictingSetters, PropertyNamer.methodToProperty(m.getName()), m));
    // 解决setting方法冲突
    resolveSetterConflicts(conflictingSetters);
  }

private void resolveSetterConflicts(Map<String, List<Method>> conflictingSetters) {
    // 遍历所有的setting方法, 寻找最合适的setting方法, 因为子类可以复写父类方法
    for (Entry<String, List<Method>> entry : conflictingSetters.entrySet()) {
      String propName = entry.getKey();
      List<Method> setters = entry.getValue();
      Class<?> getterType = getTypes.get(propName);
      boolean isGetterAmbiguous = getMethods.get(propName) instanceof AmbiguousMethodInvoker;
      boolean isSetterAmbiguous = false;
      Method match = null;
      for (Method setter : setters) {
        // 如果和getterType类型相同, 直接使用
        if (!isGetterAmbiguous && setter.getParameterTypes()[0].equals(getterType)) {
          // should be the best match
          match = setter;
          break;
        }
        if (!isSetterAmbiguous) {
          // 选择一个更加匹配的
          match = pickBetterSetter(match, setter, propName);
          isSetterAmbiguous = match == null;
        }
      }
      if (match != null) {
        // 添加到setMethod中
        addSetMethod(propName, match);
      }
    }
  }

private Method pickBetterSetter(Method setter1, Method setter2, String property) {
    if (setter1 == null) {
      return setter2;
    }
    Class<?> paramType1 = setter1.getParameterTypes()[0];
    Class<?> paramType2 = setter2.getParameterTypes()[0];
    if (paramType1.isAssignableFrom(paramType2)) {
      return setter2;
    }
    if (paramType2.isAssignableFrom(paramType1)) {
      return setter1;
    }
    MethodInvoker invoker = new AmbiguousMethodInvoker(setter1,
        MessageFormat.format(
            "Ambiguous setters defined for property ''{0}'' in class ''{1}'' with types ''{2}'' and ''{3}''.", property,
            setter2.getDeclaringClass().getName(), paramType1.getName(), paramType2.getName()));
    setMethods.put(property, invoker);
    Type[] paramTypes = TypeParameterResolver.resolveParamTypes(setter1, type);
    setTypes.put(property, typeToClass(paramTypes[0]));
    return null;
  }
```



#### 1.6 addFields

> #addFields(Class<?> clazz) 方法，初始化 `getMethods` + `getTypes` 和 `setMethods` + `setTypes` ，通过遍历 fields 属性。实际上，它是 #addGetMethods(...) 和 #addSetMethods(...) 方法的补充，因为有些 field ，不存在对应的 setting 或 getting 方法，**所以直接使用对应的 field** ，而不是方法。

```java
private void addFields(Class<?> clazz) {
    // 获得所有的field
    Field[] fields = clazz.getDeclaredFields();
    for (Field field : fields) {
      // 如果当前field不存在与setMethods中(无setter方法) 添加到setMethods中
      if (!setMethods.containsKey(field.getName())) {
        // issue #379 - removed the check for final because JDK 1.5 allows
        // modification of final fields through reflection (JSR-133). (JGB)
        // pr #16 - final static can only be set by the classloader
        int modifiers = field.getModifiers();
        if ((!Modifier.isFinal(modifiers) || !Modifier.isStatic(modifiers))) {
          addSetField(field);
        }
      }
      // 如果当前field不存在与getMethods中(无getter方法) 添加到getMethods中
      if (!getMethods.containsKey(field.getName())) {
        addGetField(field);
      }
    }
    // 递归 处理父类
    if (clazz.getSuperclass() != null) {
      addFields(clazz.getSuperclass());
    }
  }

  private void addSetField(Field field) {
    // 判断属性是否合法
    if (isValidPropertyName(field.getName())) {
      // 添加到setMethods中
      setMethods.put(field.getName(), new SetFieldInvoker(field));
      Type fieldType = TypeParameterResolver.resolveFieldType(field, type);
      // 添加到setTypes中
      setTypes.put(field.getName(), typeToClass(fieldType));
    }
  }

  private void addGetField(Field field) {
    // 判断属性是否合法
    if (isValidPropertyName(field.getName())) {
      // 添加到getMethods中
      getMethods.put(field.getName(), new GetFieldInvoker(field));
      Type fieldType = TypeParameterResolver.resolveFieldType(field, type);
      // 添加到getTypes中
      getTypes.put(field.getName(), typeToClass(fieldType));
    }
  }

  private boolean isValidPropertyName(String name) {
    return (!name.startsWith("$") && !"serialVersionUID".equals(name) && !"class".equals(name));
  }
```



### 2. ReflectorFactory

> Reflector 工厂接口，用于创建和缓存 Reflector 对象,  所在包: org.apache.ibatis.reflection.ReflectorFactory

```java
public interface ReflectorFactory {

  /**
   * 是否缓存Reflector对象
   */
  boolean isClassCacheEnabled();

  /**
   * 设置是否缓存Reflector对象
   */
  void setClassCacheEnabled(boolean classCacheEnabled);

  /**
   * 获取 Reflector 对象
   */
  Reflector findForClass(Class<?> type);
}
```



#### 2.1 DefaultReflectorFactory

> ReflectorFactory 默认的实现类

```java
public class DefaultReflectorFactory implements ReflectorFactory {
  /**
   * 是否缓存
   */
  private boolean classCacheEnabled = true;
  /**
   * 缓存映射
   * key: 类,  value: Reflector对象
   */
  private final ConcurrentMap<Class<?>, Reflector> reflectorMap = new ConcurrentHashMap<>();

  public DefaultReflectorFactory() {
  }

  @Override
  public boolean isClassCacheEnabled() {
    return classCacheEnabled;
  }

  @Override
  public void setClassCacheEnabled(boolean classCacheEnabled) {
    this.classCacheEnabled = classCacheEnabled;
  }

  @Override
  public Reflector findForClass(Class<?> type) {
    // 如果开启缓存, 则从 reflectorMap 中获取
    if (classCacheEnabled) {
      // synchronized (type) removed see issue #461
      // 不存在则创建
      return MapUtil.computeIfAbsent(reflectorMap, type, Reflector::new);
    }
    return new Reflector(type);
  }

}

```



### 3. Invoker

> 调用者接口,  所在包: org.apache.ibatis.reflection.invoker.Invoker

```java
public interface Invoker {

  /**
   * 执行调用, 具体调用什么方法, 由子类来实现
   */
  Object invoke(Object target, Object[] args) throws IllegalAccessException, InvocationTargetException;

  /**
   * @return 类
   */
  Class<?> getType();
}
```



#### 3.1 GetFieldInvoker

> 实现 Invoker 接口，获得 Field 调用者, 所在包: org.apache.ibatis.reflection.invoker.GetFieldInvoker

```java
public class GetFieldInvoker implements Invoker {

  /**
   * field 对象
   */
  private final Field field;

  public GetFieldInvoker(Field field) {
    this.field = field;
  }

  /**
   * 获得属性
   */
  @Override
  public Object invoke(Object target, Object[] args) throws IllegalAccessException {
    try {
      return field.get(target);
    } catch (IllegalAccessException e) {
      if (Reflector.canControlMemberAccessible()) {
        field.setAccessible(true);
        return field.get(target);
      }
      throw e;
    }
  }

  /**
   * 返回属性类型
   */
  @Override
  public Class<?> getType() {
    return field.getType();
  }
}
```



#### 3.2 SetFieldInvoker

> 实现 Invoker 接口，设置 Field 调用者, 所在包: org.apache.ibatis.reflection.invoker.SetFieldInvoker

```java
public class SetFieldInvoker implements Invoker {

  /**
   * field对象
   */
  private final Field field;

  public SetFieldInvoker(Field field) {
    this.field = field;
  }

  /**
   * 设置field属性
   */
  @Override
  public Object invoke(Object target, Object[] args) throws IllegalAccessException {
    try {
      field.set(target, args[0]);
    } catch (IllegalAccessException e) {
      if (!Reflector.canControlMemberAccessible()) {
        throw e;
      }
      field.setAccessible(true);
      field.set(target, args[0]);
    }
    return null;
  }

  /**
   * 返回属性类型
   */
  @Override
  public Class<?> getType() {
    return field.getType();
  }
}
```



#### 3.3 MethodInvoker

> 实现 Invoker 接口，指定方法的调用器, 所在包: org.apache.ibatis.reflection.invoker.MethodInvoker

```java
public class MethodInvoker implements Invoker {

  /**
   * 类型
   */
  private final Class<?> type;
  /**
   * 指定方法
   */
  private final Method method;

  public MethodInvoker(Method method) {
    this.method = method;
    // 参数大小为1 一般是setter方法,
    // setter方法一般会有参数设置属性的值 public void setName(String name)
    // getter方法一般都是返回属性的值 String getName()
    if (method.getParameterTypes().length == 1) {
      type = method.getParameterTypes()[0];
    } else {
      type = method.getReturnType();
    }
  }

  /**
   * 执行指定的方法
   */
  @Override
  public Object invoke(Object target, Object[] args) throws IllegalAccessException, InvocationTargetException {
    try {
      return method.invoke(target, args);
    } catch (IllegalAccessException e) {
      if (Reflector.canControlMemberAccessible()) {
        method.setAccessible(true);
        return method.invoke(target, args);
      }
      throw e;
    }
  }

  @Override
  public Class<?> getType() {
    return type;
  }
}
```



### 4. ObjectFactory

> Object 工厂接口，用于创建指定类的对象, 所在包: org.apache.ibatis.reflection.factory.ObjectFactory

```java
public interface ObjectFactory {

  /**
   * 设置Properties
   * Sets configuration properties.
   *
   * @param properties
   *          configuration properties
   */
  default void setProperties(Properties properties) {
    // NOP
  }

  /**
   * 通过默认构造器创建指定类的对象
   * Creates a new object with default constructor.
   *
   * @param <T>
   *          the generic type
   * @param type
   *          Object type
   *
   * @return the t
   */
  <T> T create(Class<T> type);

  /**
   * 通过特定的构造方法创建指定类的对象
   * Creates a new object with the specified constructor and params.
   *
   * @param <T>
   *          the generic type
   * @param type
   *          Object type
   * @param constructorArgTypes
   *          Constructor argument types
   * @param constructorArgs
   *          Constructor argument values
   *
   * @return the t
   */
  <T> T create(Class<T> type, List<Class<?>> constructorArgTypes, List<Object> constructorArgs);

  /**
   * Returns true if this object can have a set of other objects. It's main purpose is to support
   * non-java.util.Collection objects like Scala collections.
   * 判断类是否是集合
   * @param <T>
   *          the generic type
   * @param type
   *          Object type
   *
   * @return whether it is a collection or not
   *
   * @since 3.1.0
   */
  <T> boolean isCollection(Class<T> type);

}
```



#### 4.1 DefaultObjectFactory

> 实现 ObjectFactory、Serializable 接口，ObjectFactory 默认实现类, 所在包: org.apache.ibatis.reflection.factory.DefaultObjectFactory

- create

  ```java
    @Override
    public <T> T create(Class<T> type) {
      return create(type, null, null);
    }
  
    @SuppressWarnings("unchecked")
    @Override
    public <T> T create(Class<T> type, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
      // 1. 获得需要创建的类
      Class<?> classToCreate = resolveInterface(type);
      // we know types are assignable
      // 2. 创建指定类的对象
      return (T) instantiateClass(classToCreate, constructorArgTypes, constructorArgs);
    }
  
    /**
     * 返回常用的集合接口的类型
     */
    protected Class<?> resolveInterface(Class<?> type) {
      Class<?> classToCreate;
      if (type == List.class || type == Collection.class || type == Iterable.class) {
        classToCreate = ArrayList.class;
      } else if (type == Map.class) {
        classToCreate = HashMap.class;
      } else if (type == SortedSet.class) { // issue #510 Collections Support
        classToCreate = TreeSet.class;
      } else if (type == Set.class) {
        classToCreate = HashSet.class;
      } else {
        classToCreate = type;
      }
      return classToCreate;
    }
  
    private <T> T instantiateClass(Class<T> type, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
      try {
        Constructor<T> constructor;
        // 1. 通过无参构造方法, 创建指定类的对象
        if (constructorArgTypes == null || constructorArgs == null) {
          constructor = type.getDeclaredConstructor();
          try {
            return constructor.newInstance();
          } catch (IllegalAccessException e) {
            if (Reflector.canControlMemberAccessible()) {
              constructor.setAccessible(true);
              return constructor.newInstance();
            }
            throw e;
          }
        }
        // 2. 使用特定构造方法, 创建指定类的对象
        constructor = type.getDeclaredConstructor(constructorArgTypes.toArray(new Class[0]));
        try {
          return constructor.newInstance(constructorArgs.toArray(new Object[0]));
        } catch (IllegalAccessException e) {
          if (Reflector.canControlMemberAccessible()) {
            constructor.setAccessible(true);
            return constructor.newInstance(constructorArgs.toArray(new Object[0]));
          }
          throw e;
        }
      } catch (Exception e) {
        // 拼接 argValues 用于报错
        String argTypes = Optional.ofNullable(constructorArgTypes).orElseGet(Collections::emptyList).stream()
            .map(Class::getSimpleName).collect(Collectors.joining(","));
        String argValues = Optional.ofNullable(constructorArgs).orElseGet(Collections::emptyList).stream()
            .map(String::valueOf).collect(Collectors.joining(","));
        throw new ReflectionException("Error instantiating " + type + " with invalid types (" + argTypes + ") or values ("
            + argValues + "). Cause: " + e, e);
      }
    }
  ```

- isCollection

  ```java
    @Override
    public <T> boolean isCollection(Class<T> type) {
      return Collection.class.isAssignableFrom(type);
    }
  ```



### 5. Property 工具类

> 提供了 PropertyCopier、PropertyNamer、PropertyTokenizer 三个属性相关的工具类, 所在包: org.apache.ibatis.reflection.property



#### 5.1 PropertyCopier

> 属性复制器

```java
public final class PropertyCopier {

  private PropertyCopier() {
    // Prevent Instantiation of Static Class
  }

  /**
   * 将 sourceBean的属性, 复制到destinationBean中
   */
  public static void copyBeanProperties(Class<?> type, Object sourceBean, Object destinationBean) {
    // 循环, 从当前类开始, 不断复制到父类, 知道父类不存在
    Class<?> parent = type;
    while (parent != null) {
      // 获得当前 parent 类定义的属性
      final Field[] fields = parent.getDeclaredFields();
      for (Field field : fields) {
        try {
          try {
            // 复制属性
            field.set(destinationBean, field.get(sourceBean));
          } catch (IllegalAccessException e) {
            if (!Reflector.canControlMemberAccessible()) {
              throw e;
            }
            // 如果属性不可访问, 设置属性可访问
            field.setAccessible(true);
            // 复制属性
            field.set(destinationBean, field.get(sourceBean));
          }
        } catch (Exception e) {
          // Nothing useful to do, will only fail on final fields, which will be ignored.
        }
      }
      // 获取父类
      parent = parent.getSuperclass();
    }
  }

}

```



#### 5.2 PropertyNamer

> 属性名相关的工具类方法

```java
public final class PropertyNamer {

  private PropertyNamer() {
    // Prevent Instantiation of Static Class
  }

  /**
   * 根据方法名，获得对应的属性名
   */
  public static String methodToProperty(String name) {
    // is 方法
    if (name.startsWith("is")) {
      name = name.substring(2);
      // get / set方法
    } else if (name.startsWith("get") || name.startsWith("set")) {
      name = name.substring(3);
      // 非 is get set方法抛出异常
    } else {
      throw new ReflectionException(
        "Error parsing property name '" + name + "'.  Didn't start with 'is', 'get' or 'set'.");
    }
    // 首字母小写
    if (name.length() == 1 || name.length() > 1 && !Character.isUpperCase(name.charAt(1))) {
      name = name.substring(0, 1).toLowerCase(Locale.ENGLISH) + name.substring(1);
    }

    return name;
  }

  /**
   * 判断是否为 get set方法
   */
  public static boolean isProperty(String name) {
    return isGetter(name) || isSetter(name);
  }

  /**
   * 判断是否是get is方法
   */
  public static boolean isGetter(String name) {
    return name.startsWith("get") && name.length() > 3 || name.startsWith("is") && name.length() > 2;
  }

  /**
   * 判断是否为 set方法
   */
  public static boolean isSetter(String name) {
    return name.startsWith("set") && name.length() > 3;
  }

}
```



#### 5.3 PropertyTokenizer

> 实现 Iterator 接口，属性分词器，支持迭代器的访问方式, 所在包: org.apache.ibatis.reflection.property.PropertyTokenizer

```java
public class PropertyTokenizer implements Iterator<PropertyTokenizer> {

  /**
   * 当前字符串
   */
  private String name;
  /**
   * 索引的 {@link #name} ，因为 {@link #name} 如果存在 {@link #index} 会被更改
   */
  private final String indexedName;
  /**
   * 索引
   */
  private String index;
  /**
   * 剩余字符串
   */
  private final String children;

  public PropertyTokenizer(String fullname) {
    // 初始化 name children字符串, 使用 . 作为分隔符
    int delim = fullname.indexOf('.');
    if (delim > -1) {
      name = fullname.substring(0, delim);
      children = fullname.substring(delim + 1);
    } else {
      name = fullname;
      children = null;
    }
    // 记录当前name
    indexedName = name;
    // 若存在 [, 则获得index, 并修改name
    delim = name.indexOf('[');
    if (delim > -1) {
      index = name.substring(delim + 1, name.length() - 1);
      name = name.substring(0, delim);
    }
  }

  public String getName() {
    return name;
  }

  public String getIndex() {
    return index;
  }

  public String getIndexedName() {
    return indexedName;
  }

  public String getChildren() {
    return children;
  }

  /**
   * 判断是否有下一个元素
   */
  @Override
  public boolean hasNext() {
    return children != null;
  }

  @Override
  public PropertyTokenizer next() {
    // 迭代获得下一个 PropertyTokenizer 对象
    return new PropertyTokenizer(children);
  }

  @Override
  public void remove() {
    throw new UnsupportedOperationException(
        "Remove is not supported, as it has no meaning in the context of properties.");
  }
}
```



### 6. MetaClass

> 类的元数据，基于 Reflector 和 PropertyTokenizer ，提供对指定类的各种操作, 所在包: org.apache.ibatis.reflection.MetaClass

- 构造方法

  ```java
    private final ReflectorFactory reflectorFactory;
    private final Reflector reflector;
  
    private MetaClass(Class<?> type, ReflectorFactory reflectorFactory) {
      this.reflectorFactory = reflectorFactory;
      this.reflector = reflectorFactory.findForClass(type);
    }
  
    /**
     * 静态方法, 创建指定类的 MetaClass 对象
     */
    public static MetaClass forClass(Class<?> type, ReflectorFactory reflectorFactory) {
      return new MetaClass(type, reflectorFactory);
    }
  
    /**
     * 创建类的指定属性的类的 MetaClass 对象
     */
    public MetaClass metaClassForProperty(String name) {
      Class<?> propType = reflector.getGetterType(name);
      return MetaClass.forClass(propType, reflectorFactory);
    }
  ```

- findProperty

  ```java
    /**
     * 根据表达式，获得属性
     */
    public String findProperty(String name, boolean useCamelCaseMapping) {
      // 下划线转驼峰, 为什么转空串?
      if (useCamelCaseMapping) {
        name = name.replace("_", "");
      }
      return findProperty(name);
    }
  
    public String findProperty(String name) {
      // 构建属性
      StringBuilder prop = buildProperty(name, new StringBuilder());
      return prop.length() > 0 ? prop.toString() : null;
    }
  
    private StringBuilder buildProperty(String name, StringBuilder builder) {
      // 创建 PropertyTokenizer 对象，对 name 进行分词
      PropertyTokenizer prop = new PropertyTokenizer(name);
      // 有子表达式
      if (prop.hasNext()) {
        // 获得属性名, 并添加到 builder 中
        String propertyName = reflector.findPropertyName(prop.getName());
        if (propertyName != null) {
          // 拼接属性到builder中
          builder.append(propertyName);
          builder.append(".");
          // 创建 MetaClass 对象
          MetaClass metaProp = metaClassForProperty(propertyName);
          // 递归解析子表达式 children, 并将结果添加到builder中
          metaProp.buildProperty(prop.getChildren(), builder);
        }
        // 无子表达式
      } else {
        // 获得属性名, 并添加到builder中
        String propertyName = reflector.findPropertyName(name);
        if (propertyName != null) {
          builder.append(propertyName);
        }
      }
      return builder;
    }
  
   /**
     * 通过该方法 完成驼峰转下划线
     */
    public String findPropertyName(String name) {
      return caseInsensitivePropertyMap.get(name.toUpperCase(Locale.ENGLISH));
    }
  ```

- hasGetter

  ```java
    public boolean hasGetter(String name) {
      // 生成 PropertyTokenizer 对象, 对name进行分词
      PropertyTokenizer prop = new PropertyTokenizer(name);
      // 无子表达式
      if (!prop.hasNext()) {
        // 判断是否有属性getting方法
        return reflector.hasGetter(prop.getName());
      }
      // 判断是否有getter方法
      if (reflector.hasGetter(prop.getName())) {
        // 有则创建 MetaClass 对象
        MetaClass metaProp = metaClassForProperty(prop);
        return metaProp.hasGetter(prop.getChildren());
      }
      return false;
    }
  
    private MetaClass metaClassForProperty(PropertyTokenizer prop) {
      // 调用getting方法获得返回类型
      Class<?> propType = getGetterType(prop);
      // 创建MetaClass对象
      return MetaClass.forClass(propType, reflectorFactory);
    }
  
    private Class<?> getGetterType(PropertyTokenizer prop) {
      // 获得返回类型
      Class<?> type = reflector.getGetterType(prop.getName());
      // 如果获得数组某个位置的元素, 则获取其泛型 list[0].field 可以反推list是什么类型的数组
      if (prop.getIndex() != null && Collection.class.isAssignableFrom(type)) {
        // 获得返回的类型
        Type returnType = getGenericGetterType(prop.getName());
        if (returnType instanceof ParameterizedType) {
          Type[] actualTypeArguments = ((ParameterizedType) returnType).getActualTypeArguments();
          if (actualTypeArguments != null && actualTypeArguments.length == 1) {
            returnType = actualTypeArguments[0];
            if (returnType instanceof Class) {
              type = (Class<?>) returnType;
            } else if (returnType instanceof ParameterizedType) {
              type = (Class<?>) ((ParameterizedType) returnType).getRawType();
            }
          }
        }
      }
      return type;
    }
  
    private Type getGenericGetterType(String propertyName) {
      try {
        // 获得 Invoker 对象
        Invoker invoker = reflector.getGetInvoker(propertyName);
        // 如果是 MethodInvoker对象, 则是getting方法, 解析方法返回类型
        if (invoker instanceof MethodInvoker) {
          Field declaredMethod = MethodInvoker.class.getDeclaredField("method");
          declaredMethod.setAccessible(true);
          Method method = (Method) declaredMethod.get(invoker);
          return TypeParameterResolver.resolveReturnType(method, reflector.getType());
        }
        // GetFieldInvoker 对象, 说明是field. 直接访问
        if (invoker instanceof GetFieldInvoker) {
          Field declaredField = GetFieldInvoker.class.getDeclaredField("field");
          declaredField.setAccessible(true);
          Field field = (Field) declaredField.get(invoker);
          return TypeParameterResolver.resolveFieldType(field, reflector.getType());
        }
      } catch (NoSuchFieldException | IllegalAccessException e) {
        // Ignored
      }
      return null;
    }
  ```

- getGetterType

  ```java
    public Class<?> getGetterType(String name) {
      // 创建 PropertyTokenizer 对象, 对name进行分词
      PropertyTokenizer prop = new PropertyTokenizer(name);
      // 有子表达式
      if (prop.hasNext()) {
        // 获取MetaClass对象
        MetaClass metaProp = metaClassForProperty(prop);
        // 递归判断子表达式, 获得返回值的类型
        return metaProp.getGetterType(prop.getChildren());
      }
      // issue #506. Resolve the type inside a Collection Object
      // 没有子表达式, 直接获取返回值的类型
      return getGetterType(prop);
    }
  ```



### 7. ObjectWrapper

> 对象包装器接口，基于 MetaClass 工具类，定义对指定对象的各种操作 所在包: org.apache.ibatis.reflection.wrapper.ObjectWrapper

```java
  public interface ObjectWrapper {

  /**
   * 获得值
   */
  Object get(PropertyTokenizer prop);

  /**
   * 设置值
   */
  void set(PropertyTokenizer prop, Object value);

  /**
   * {@link MetaClass#findProperty(String, boolean)}
   */
  String findProperty(String name, boolean useCamelCaseMapping);

  /**
   * {@link MetaClass#getGetterNames()}
   */
  String[] getGetterNames();

  /**
   * {@link MetaClass#getSetterNames()}
   */
  String[] getSetterNames();

  /**
   * {@link MetaClass#getSetterType(String)}
   */
  Class<?> getSetterType(String name);

  /**
   * {@link MetaClass#getGetterType(String)}
   */
  Class<?> getGetterType(String name);

  /**
   * {@link MetaClass#hasSetter(String)}
   */
  boolean hasSetter(String name);

  /**
   * {@link MetaClass#hasGetter(String)}
   */
  boolean hasGetter(String name);

  /**
   * {@link MetaObject#forObject(Object, ObjectFactory, ObjectWrapperFactory, ReflectorFactory)}
   */
  MetaObject instantiatePropertyValue(String name, PropertyTokenizer prop, ObjectFactory objectFactory);

  /**
   * 是否为集合
   */
  boolean isCollection();

  /**
   * 添加元素到集合
   */
  void add(Object element);

  /**
   * 添加多个元素到集合
   */
  <E> void addAll(List<E> element);

}
```



#### 7.1 BaseWrapper

> 实现 ObjectWrapper 接口，ObjectWrapper 抽象类，为子类 BeanWrapper 和 MapWrapper 提供属性值的获取和设置的公用方法, 所在包: org.apache.ibatis.reflection.wrapper.BaseWrapper

```java
public abstract class BaseWrapper implements ObjectWrapper {

  protected static final Object[] NO_ARGUMENTS = {};
  /**
   * MetaObject 对象
   */
  protected final MetaObject metaObject;

  protected BaseWrapper(MetaObject metaObject) {
    this.metaObject = metaObject;
  }

  /**
   * 获得指定属性的值
   */
  protected Object resolveCollection(PropertyTokenizer prop, Object object) {
    if ("".equals(prop.getName())) {
      return object;
    }
    return metaObject.getValue(prop.getName());
  }

  /**
   * 获得集合中指定位置的值
   */
  protected Object getCollectionValue(PropertyTokenizer prop, Object collection) {
    if (collection == null) {
      throw new ReflectionException("Cannot get the value '" + prop.getIndexedName() + "' because the property '"
          + prop.getName() + "' is null.");
    }
    if (collection instanceof Map) {
      return ((Map) collection).get(prop.getIndex());
    }
    int i = Integer.parseInt(prop.getIndex());
    if (collection instanceof List) {
      return ((List) collection).get(i);
    } else if (collection instanceof Object[]) {
      return ((Object[]) collection)[i];
    } else if (collection instanceof char[]) {
      return ((char[]) collection)[i];
    } else if (collection instanceof boolean[]) {
      return ((boolean[]) collection)[i];
    } else if (collection instanceof byte[]) {
      return ((byte[]) collection)[i];
    } else if (collection instanceof double[]) {
      return ((double[]) collection)[i];
    } else if (collection instanceof float[]) {
      return ((float[]) collection)[i];
    } else if (collection instanceof int[]) {
      return ((int[]) collection)[i];
    } else if (collection instanceof long[]) {
      return ((long[]) collection)[i];
    } else if (collection instanceof short[]) {
      return ((short[]) collection)[i];
    } else {
      throw new ReflectionException("Cannot get the value '" + prop.getIndexedName() + "' because the property '"
          + prop.getName() + "' is not Map, List or Array.");
    }
  }


  /**
   * 设置集合中指定位置的值
   */
  protected void setCollectionValue(PropertyTokenizer prop, Object collection, Object value) {
    if (collection == null) {
      throw new ReflectionException("Cannot set the value '" + prop.getIndexedName() + "' because the property '"
          + prop.getName() + "' is null.");
    }
    if (collection instanceof Map) {
      ((Map) collection).put(prop.getIndex(), value);
    } else {
      int i = Integer.parseInt(prop.getIndex());
      if (collection instanceof List) {
        ((List) collection).set(i, value);
      } else if (collection instanceof Object[]) {
        ((Object[]) collection)[i] = value;
      } else if (collection instanceof char[]) {
        ((char[]) collection)[i] = (Character) value;
      } else if (collection instanceof boolean[]) {
        ((boolean[]) collection)[i] = (Boolean) value;
      } else if (collection instanceof byte[]) {
        ((byte[]) collection)[i] = (Byte) value;
      } else if (collection instanceof double[]) {
        ((double[]) collection)[i] = (Double) value;
      } else if (collection instanceof float[]) {
        ((float[]) collection)[i] = (Float) value;
      } else if (collection instanceof int[]) {
        ((int[]) collection)[i] = (Integer) value;
      } else if (collection instanceof long[]) {
        ((long[]) collection)[i] = (Long) value;
      } else if (collection instanceof short[]) {
        ((short[]) collection)[i] = (Short) value;
      } else {
        throw new ReflectionException("Cannot set the value '" + prop.getIndexedName() + "' because the property '"
            + prop.getName() + "' is not Map, List or Array.");
      }
    }
  }

  protected Object getChildValue(PropertyTokenizer prop) {
    MetaObject metaValue = metaObject.metaObjectForProperty(prop.getIndexedName());
    if (metaValue == SystemMetaObject.NULL_META_OBJECT) {
      return null;
    }
    return metaValue.getValue(prop.getChildren());
  }

  protected void setChildValue(PropertyTokenizer prop, Object value) {
    MetaObject metaValue = metaObject.metaObjectForProperty(prop.getIndexedName());
    if (metaValue == SystemMetaObject.NULL_META_OBJECT) {
      if (value == null) {
        // don't instantiate child path if value is null
        return;
      }
      metaValue = instantiatePropertyValue(null, new PropertyTokenizer(prop.getName()), metaObject.getObjectFactory());
    }
    metaValue.setValue(prop.getChildren(), value);
  }
}
```



#### 7.2 BeanWrapper

> 继承 BaseWrapper 抽象类，**普通对象**的 ObjectWrapper 实现类，例如 User、Order 这样的 POJO 类, 所在包: org.apache.ibatis.reflection.wrapper.BeanWrapper

- 构造方法

  ```java
    /**
     * 普通对象
     */
    private final Object object;
    private final MetaClass metaClass;
  
    public BeanWrapper(MetaObject metaObject, Object object) {
      super(metaObject);
      this.object = object;
      // 创建MetaClass对象
      this.metaClass = MetaClass.forClass(object.getClass(), metaObject.getReflectorFactory());
    }
  ```

- get

  ```java
    @Override
    public Object get(PropertyTokenizer prop) {
      // 如果还有子列
      if (prop.hasNext()) {
        return getChildValue(prop);
        // 获得集合类型的属性的指定位置的值
      } else if (prop.getIndex() != null) {
        // 获得指定位置的值
        return getCollectionValue(prop, resolveCollection(prop, object));
      } else {
        // 获得属性的值
        return getBeanProperty(prop, object);
      }
    }
  
    private Object getBeanProperty(PropertyTokenizer prop, Object object) {
      try {
        Invoker method = metaClass.getGetInvoker(prop.getName());
        try {
          return method.invoke(object, NO_ARGUMENTS);
        } catch (Throwable t) {
          throw ExceptionUtil.unwrapThrowable(t);
        }
      } catch (RuntimeException e) {
        throw e;
      } catch (Throwable t) {
        throw new ReflectionException(
            "Could not get property '" + prop.getName() + "' from " + object.getClass() + ".  Cause: " + t.toString(), t);
      }
    }
  ```

- set

  ```java
    @Override
    public void set(PropertyTokenizer prop, Object value) {
      // 还有子列需要设置值
      if (prop.hasNext()) {
        setChildValue(prop, value);
        // 获得集合类型的属性
      } else if (prop.getIndex() != null) {
        setCollectionValue(prop, resolveCollection(prop, object), value);
        // 设置属性的值
      } else {
        setBeanProperty(prop, object, value);
      }
    }
  
    private void setBeanProperty(PropertyTokenizer prop, Object object, Object value) {
      try {
        Invoker method = metaClass.getSetInvoker(prop.getName());
        Object[] params = { value };
        try {
          method.invoke(object, params);
        } catch (Throwable t) {
          throw ExceptionUtil.unwrapThrowable(t);
        }
      } catch (Throwable t) {
        throw new ReflectionException("Could not set property '" + prop.getName() + "' of '" + object.getClass()
            + "' with value '" + value + "' Cause: " + t.toString(), t);
      }
    }
  ```

  
