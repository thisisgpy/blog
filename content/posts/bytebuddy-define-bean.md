---
title: "使用 ByteBuddy 动态生成 EasyExcel 数据映射 Bean"
date: 2022-05-18T15:21:13+08:00
draft: false
tags: ["Tech"]
slug: "bytebuddy-easyexcel"
---

阿里的 EasyExcel 可以基于一个 Java Bean 来方便的实现数据的读取和写出，但是 Excel 表头的变更都会引起这个 Java Bean 源码的变动。为了能够动态适应 Excel 表头的变更，本文使用 ByteBuddy 工具类来动态生成 Java Bean，以避免源码级的改动。

使用 ByteBuddy 动态创建一个类的最简单方法就是创建一个 `Object` 的子类：

```java
DynamicType.Builder<Object> builder = new ByteBuddy()
          .subclass(Object.class)
          .name("io.buyan.demo.beans.DemoBean");
```

这时候 `builder` 就可以简单理解为 ByteBuddy 创建的这个类的字节码，之后所有的操作都基于 `builder` 进行。由于 ByteBuddy 将 `builder` 设计为了不可变模式，所以如果不是链式调用相关 API，那么对 `builder` 的每一次操作都需要接收其返回的新的 `builder` 实例。

Excel 表头的变更意味着 Java Bean 属性定义的变更，所以就需要将 Java Bean 的属性定义存储在某个地方，数据库、文本、配置文件都可以。每次动态创建这个 Java Bean 时都需要先去读取这些配置。

```java
@Data
public class EasyExcelMappingFieldDefinition {
    /**
     * 属性名称
     */
    private String fieldName;
    /**
     * 属性类型
     */
    private Class<?> fieldType;
  	/**
     * 表头名称
     */
    private String headerName;
}

/**
  * 从数据库读取 Java Bean 的属性定义
  */
public List<EasyExcelMappingFieldDefinition> getDefinitionFromDB() {
  List<EasyExcelMappingFieldDefinition> definitions = new ArrayList<>();
  definitions.add(new EasyExcelMappingFieldDefinition("name", String.class, "姓名"));
  definitions.add(new EasyExcelMappingFieldDefinition("age", int.class, "年龄"));
  return definitions;
}

```

拿到这些属性定义后，使用 ByteBuddy 将属性写入动态创建的类即可。

```java
List<EasyExcelMappingFieldDefinition> definitions = fieldDefinitionService.getDefinitionFromDB();
for (EasyExcelMappingFieldDefinition definition : definitions) {
  // 注意 builder 是不可变模式
  builder = builder.defineProperty(definition.getFieldName(), definition.getFieldType());
}
```

`defineProperty()` 方法会自动为属性生成 `getter/setter`，便于数据存取。ByteBuddy 还提供了 `defineField()` 方法来定义一个属性，但不会自动生成 `getter/setter`。

这里只是定义了 Java Bean 的属性，但是 EasyExcel 的工作是需要读取属性上的 `@ExcelProperty` 注解来完成，所以还要为每一个属性创建这个注解。在 ByteBuddy 中，使用 `AnnotationDescription` 来描述一个注解。

```java
AnnotationDescription annotation = AnnotationDescription.Builder.ofType(ExcelProperty.class)
                    .define("value", definition.getHeaderName())
                    .build();
```

如果需要为注解的多个属性指定值，可以链式调用 `define()` 方法。注解定义完成后，使用 `builder` 将注解添加到对应的属性上即可。

```java
for (EasyExcelMappingFieldDefinition definition : definitions) {
  // 创建注解
  AnnotationDescription annotation = AnnotationDescription.Builder.ofType(ExcelProperty.class)
    .define("value", definition.getHeaderName())
    .build();
  builder = builder.defineProperty(definition.getFieldName(), definition.getFieldType())
    .annotateField(annotation); // 添加注解到属性上
}
```

最后调用 ByteBuddy 的 API 完成类的编译和加载即可拿到这个动态创建的 Java Baen 的 `Class` 对象，加载时可以根据实际的业务场景来指定用于类加载的 ClassLoader。

```java
Class<?> beanClass = builder.make() // 编译
  .load(Thread.currentThread().getContextClassLoader()) // 加载
  .getLoaded(); // 获取 Class
```

最后拿到的这个 `Class` 可以直接在 EasyExcel 的 API 中使用。但由于这个类是动态生成的，EasyExcel 的 `ReadListener` 的泛型就无法指定一个明确的类型，当 `invoke()` 方法调用时，拿到的数据行对象会是一个 Object，对于数据的处理就只能借助反射 API 来完成，这算是此方案最大的一个缺点。
