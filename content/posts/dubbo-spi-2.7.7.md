---
title: "Dubbo SPI 机制 - 基于 2.7.7 版本"
date: 2022-03-30T17:52:35+08:00
draft: false
tags: ["Tech"]
slug: "dubbo-spi"
---

Dubbo SPI 机制涉及到 `@SPI`、`@Adaptive`、`@Activate` 三个注解，ExtensionLoader 作为 Dubbo SPI 机制的核心负责加载和管理扩展点及其实现。本文以 ExtensionLoader 的源码作为分析主线，进而引出三个注解的作用和工作机制。

ExtensionLoader 被设计为只能通过 `getExtensionLoader(Class<T> type)` 方法获取到实例，参数 `type` 表示拿到的这个实例要负责加载的扩展点类型。为了避免在之后的源码分析中产生困惑，请先记住这个结论：**每个 ExtensionLoader 只能加载其绑定的扩展点类型（即 type 的类型）的具体实现**。也就是说，如果 `type` 的值是 `Protocol.class`，那么这个 ExtensionLoader 的实例就只能加载 Protocol 接口的实现，不能去加载 Compiler 接口的实现。

# 怎么获取扩展实现

在 Dubbo 里，如果一个接口标注了 `@SPI` 注解，那么它就表示一个扩展点类型，这个接口的实现就是这个扩展点的实现。比如 Protocol 接口的声明：

```java
@SPI("dubbo")
public interface Protocol {}
```

一个扩展点可能存在多个实现，可以使用 `@SPI` 注解的 `value` 属性指定要选择的默认实现。当用户没有明确指定要使用哪个实现时，Dubbo 就会自动选择这个默认实现。

 `getExtension(String name)` 方法可以获取指定名称的扩展实现的实例，这个扩展实现的类型必须是当前 ExtensionLoader 绑定的扩展类型。这个方法会先查缓存里是否有这个扩展实现的实例，如果没有再通过 `createExtension(String name)` 方法创建实例。Dubbo 在这一块设置了多层缓存，进入 `createExtension(String name)` 方法后又会调用 `getExtensionClasses()` 方法拿到当前 ExtensionLoader 已加载的所有扩展实现。如果还拿不到，那就调用 `loadExtensionClasses()` 方法真的去加载了。

```java
private Map<String, Class<?>> loadExtensionClasses() {
  // 取 @SPI 注解上的值（只允许存在一个值）保存到 cachedDefaultName
  cacheDefaultExtensionName();
  Map<String, Class<?>> extensionClasses = new HashMap<>();
  // 不同的策略代表不同的目录，迭代进行加载
  for (LoadingStrategy strategy : strategies) {
    // loadDirectory(...)
    // 执行不同策略
  }
  return extensionClasses;
}
```

 `cacheDefaultExtensionName()` 方法会从当前 ExtensionLoader 绑定的 `type` 上去获取 `@SPI` 注解，并将其 `value` 值保存到 ExtensionLoader 的 `cachedDefaultName` 字段用来表示扩展点的默认扩展实现的名称。

## SPI 配置的加载策略

接着迭代三种扩展实现加载策略。`strategies` 是通过 `loadLoadingStrategies()` 方法加载的，在这个方法里已经对三种策略进行了优先级排序，排序规则是**低优先级的策略放在前面**。简单看一下 LoadingStrategy 接口：

```java
public interface LoadingStrategy extends Prioritized {
    String directory();
    default boolean preferExtensionClassLoader() {
        return false;
    }
    default String[] excludedPackages() {
        return null;
    }
    default boolean overridden() {
        return false;
    }
}
```

`overridden()` 方法表示当前策略加载的扩展实现是否可以覆盖比其优先级低的策略加载的扩展实现，优先级由 Prioritized 接口控制。为了在加载扩展实现时能够方便的进行覆盖操作，对加载策略进行预先排序就非常重要。这也是 `loadLoadingStrategies()` 方法要排序的原因。

## 查找和解析 SPI 配置文件

`loadDirectory()` 方法在当前策略指定的目录下查找 SPI 配置文件并加载为 `java.net.URL` 对象，接下来 `loadResource()` 方法对配置文件进行逐行解析。Dubbo SPI 的配置文件是 `key=value` 形式，`key` 表示扩展实现的名称，`value` 是扩展实现的具体类名，这里直接 `split` 后对扩展实现进行加载，最后交给 `loadClass()` 方法处理。

```java
private void loadClass(Map<String, Class<?>> extensionClasses, java.net.URL resourceURL, Class<?> clazz, String name, boolean overridden) throws NoSuchMethodException {
  if (!type.isAssignableFrom(clazz)) {
    throw new IllegalStateException("...");
  }
  // 适配类
  if (clazz.isAnnotationPresent(Adaptive.class)) {
    cacheAdaptiveClass(clazz, overridden);
  } else if (isWrapperClass(clazz)) { // 包装类
    cacheWrapperClass(clazz);
  } else {
    clazz.getConstructor(); // 检查点：扩展类必须要有一个无参构造器
    // 兜底策略：如果配置文件没有按 key=value 这样写，就取类的简单名称作为 key，即 name
    if (StringUtils.isEmpty(name)) {
      name = findAnnotationName(clazz);
      if (name.length() == 0) {
        throw new IllegalStateException("..." + resourceURL);
      }
    }

    String[] names = NAME_SEPARATOR.split(name);
    if (ArrayUtils.isNotEmpty(names)) {
      // 如果当前实现类标注了 @Activate 则缓存
      cacheActivateClass(clazz, names[0]);
      // 扩展实现可以用逗号分隔取很多名字（a,b,c=com.xxx.Yyy），这里迭代所有名字做缓存
      for (String n : names) {
        // 缓存 扩展实现的实例 -> 名称
        cacheName(clazz, n);
        // 缓存 名称 -> 扩展实现的实例
        saveInExtensionClass(extensionClasses, clazz, n, overridden);
      }
    }
  }
}
```

`cacheAdaptiveClass()` 方法是对 `@Adaptive` 的处理，这个稍后会介绍。

## 包装类

来看 `isWrapperClass()` 方法，这个方法用来判断当前实例化的扩展实现是否为包装类。判断条件非常简单，**只要某个类具有一个只有一个参数的构造器，且这个参数的类型和当前 ExtensionLoader 绑定的扩展类型一致，这个类就是包装类**。

在 Dubbo 中包装类都是以 Wrapper 结尾，比如 QosProtocolWrapper：

```java
public class QosProtocolWrapper implements Protocol {
  private Protocol protocol;
	// 包装类必要的构造器
  public QosProtocolWrapper(Protocol protocol) {
    if (protocol == null) {
      throw new IllegalArgumentException("protocol == null");
    }
    this.protocol = protocol;
  }
  
  @Override
    public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
        if (UrlUtils.isRegistry(invoker.getUrl())) { // 一些额外的逻辑
            startQosServer(invoker.getUrl());
            return protocol.export(invoker);
        }
        return protocol.export(invoker);
    }

    @Override
    public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
        if (UrlUtils.isRegistry(url)) { // 一些额外的逻辑
            startQosServer(url);
            return protocol.refer(type, url);
        }
        return protocol.refer(type, url);
    }
}
```

可以看到，Dubbo 中的包装类实际上就是 AOP 的一种实现，并且多个包装类可以不断嵌套，类似 Java I/O 类库的设计。回到 `loadClass()` 方法，如果当前是包装类，则放入 `cachedWrapperClasses` 集合中保存。

## 兜底不标准的 SPI 配置文件

在 `loadClass()` 方法的最后一个 `else` 分支中，首先去获取了一次当前扩展实现的无参构造器，因为之后实例化扩展实现的时候需要这个构造器，这里等于是提前做了一个检查。然后是做兜底操作，因为 SPI 配置文件可能没有按照 Dubbo 的要求写成 `key=value` 形式，那么就把扩展实现类的类名作为 `key`。`cacheActivateClass()`  方法用于判断当前扩展实现是否携带了 `@Activate` 注解，如果有则缓存，这个注解的用处后文会详述。

## 扩展实现及其名称的多种缓存

最后把扩展实现的名称和扩展实现的 Class 对象进行双向缓存。`cacheName()` 方法做 Class 对象到扩展实现名称的映射，`saveInExtensionClass()` 是做扩展实现名称到 Class 对象的映射。

`saveInExtensionClass()` 方法的参数 `overridden` 实际就是来自于加载策略 LoadingStrategy 的 `overridden()` 方法。上文提到过三个加载策略是在迭代时是按照优先级从小到大顺序进行的，所以只要当前的 LoadingStrategy 允许覆盖之前策略创建的扩展实现，那么这里 `overridden` 就为 `true`。

到了这里实际上就是 `loadExtensionClasses()` 方法的全部执行逻辑，当方法执行完成后当前 ExtensionLoader 所绑定的扩展类型的所有实现类就全部被加载成了 Class 对象并放入了 `cachedClasses` 中。

## 实例化扩展实现

再往上返回到 `createExtension(String name)` 中，如果在已加载的扩展实现类里找不到当前要获取扩展实现则抛出异常。接着尝试从缓存中获取一下对应的实例，如果没有则实例化并放入缓存。`injectExtension()` 方法就是通过反射将当前实例化出来的扩展实现所依赖的其他扩展实现也初始化并赋值。

这里用到一个 `ExtensionFactory objectFactory`，AdaptiveExtensionFactory 作为 ExtensionFactory 的适配实现，对 SpiExtensionFactory 和 SpringExtensionFactory 进行了适配。当要获取一个扩展实现时，都是调用 AdaptiveExtensionFactory 的 `getExtension(Class<T> type, String name)` 方法。

```java
public <T> T getExtension(Class<T> type, String name) {
  for (ExtensionFactory factory : factories) {
    T extension = factory.getExtension(type, name);
    if (extension != null) {
      return extension;
    }
  }
  return null;
}
```

这个方法分别尝试调用两个具体实现的 `getExtension()` 方法来获取扩展实现。SpiExtensionFactory 是从 Dubbo 自己的容器里查找扩展实现，实际就是调用 ExtensionLoader 的方法来实现，算是一个门面。SpringExtensionFactory 顾名思义就是从 Spring 容器内查找扩展实现，毕竟很多时候 Dubbo 都是配合着 Spring 在使用。

回到 `createExtension(String name)` 方法继续往下看，接下来是迭代在加载扩展实现时保存的包装类，滚动将上一个包装完的实例作为下一个包装类的构造器参数进行包装，也就是说最终拿到的扩展实现的实例是最后一个包装类的实例。最后的最后，如果扩展实现有 Lifecycle 接口，则调用其 `initialize()` 方法初始化生命周期。至此，一个扩展实现就被创建出来了！

# 怎么选择要使用的扩展实现

在 `loadClass()` 方法中提到过，如果加载的扩展实现带有 `@Adaptive` 注解，`cacheAdaptiveClass()` 方法将会把这个扩展实现按照加载策略的覆盖（overridden）设置赋值给 `cachedAdaptiveClass`。

## @Adaptive 的作用

Dubbo 中的扩展点一般都具有很多个扩展实现，简单说就是一个接口存在很多个实现。但接口是不能被实例化的，所以要在运行时找一个具体的实现类来实例化。 `@Adaptive` 是用来在运行时决定选择哪个实现的。如果标注在类上就表示这个类是适配类，加载扩展实现的时候直接赋值给 ExtensionLoader 的 `cachedAdaptiveClass` 字段即可，例如上文讲到的 AdaptiveExtensionFactory。

所以这里简单总结一下，所谓**适配类就是在实际使用扩展点的时候用来选择具体的扩展实现的那个类**。

`@Adaptive` 也可以标注在接口方法上，表示这个方法要在运行时通过字节码生成工具动态生成方法体，在方法体内选择具体的实现来使用，比如 Protocol 接口：

```java
@SPI("dubbo")
public interface Protocol {
  @Adaptive
  <T> Exporter<T> export(Invoker<T> invoker) throws RpcException;
  @Adaptive
  <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException;
}
```

很明显，Protocol 的每个实现都有自己暴露服务和引用服务的逻辑，如果直接根据 URL 去解析要使用的协议并实例化显然不是一个好的选择。作为一个 Spring 应用工程师，应该立刻想到 IoC 才是人间正道。Dubbo 的开发者（可能）也是这么想的，但是自己搞一套 IoC 出来又好像不是太合适，于是就通过了字节码增强的方式来实现。

## 动态适配类的创建

如果一个扩展点的所有实现类上都没有携带 `@Adaptive` 注解，但是扩展点的某些方法上带了 `@Adaptive` 注解，这就表示 Dubbo 需要在运行时使用字节码增强工具动态的创建一个扩展点的代理类，在代理类的同名方法里选择具体的扩展实现进行调用。

这么说有点抽象，我们来看 ExtensionLoader 的 `getAdaptiveExtension()` 方法。这个方法获取当前 ExtensionLoader 绑定的扩展点的适配类，首先从 `cachedAdaptiveInstance` 上尝试获取，这个字段保存的是上文提到的 `cachedAdaptiveClass` 实例化的结果。如果获取不到，经过双重检查锁后调用 `createAdaptiveExtension()` 方法进行适配类的创建。

`createAdaptiveExtension()` 方法又调用 `getAdaptiveExtensionClass()` 方法拿到适配类的 Class 对象，即上文提到的 `cachedAdaptiveClass`，然后将 Class 实例化后调用 `injectExtension()` 方法进行注入。

`getAdaptiveExtensionClass()` 方法发现 `cachedAdaptiveClass` 没有值后转而调用 `createAdaptiveExtensionClass()` 方法动态生成一个适配类。这里涉及到的几个方法很简单就不贴代码了，下面看一下动态生成适配的方法。

```java
private Class<?> createAdaptiveExtensionClass() {
  String code = new AdaptiveClassCodeGenerator(type, cachedDefaultName).generate();
  ClassLoader classLoader = findClassLoader();
  org.apache.dubbo.common.compiler.Compiler compiler = 
    ExtensionLoader.getExtensionLoader(org.apache.dubbo.common.compiler.Compiler.class)
    .getAdaptiveExtension();
  return compiler.compile(code, classLoader);
}
```

首先调用 AdaptiveClassCodeGenerator 类的 `generate()` 方法把适配类生成好，然后也是走 SPI 机制拿到需要的 Compiler 的适配类执行编译，最后把编译出来的适配类的 Class 对象返回。

Dubbo 使用 javassist 框架来动态生成适配类，AdaptiveClassCodeGenerator 类的 `generate()` 方法实际就是做的适配类文件的字符串拼接。具体的生成逻辑没有什么好讲的，都是些字符串操作，这里简单写个示例：

```java
@SPI
interface TroubleMaker {
    @Adaptive
    Server bind(arg0, arg1);
  
    Result doSomething();
}
public class TroubleMaker$Adaptive implements TroubleMaker {
  
    public Result doSomething() {
      throw new UnsupportedOperationException("The method doSomething of interface TroubleMaker is not adaptive method!");
    }

    public Server bind(arg0, arg1) {
        TroubleMaker extension =
            (TroubleMaker) ExtensionLoader
                .getExtensionLoader(TroubleMaker.class)
                .getExtension(extName);
        return extension.bind(arg0, arg1);
    }
}
```

假设有个扩展点叫 TroubleMaker，那么动态生成的适配类就叫做 TroubleMaker$Adaptive，适配类对没有标注 `@Adaptive` 注解的方法会直接抛出异常，而使用了 `@Adaptive` 注解的方法内部实际是通过 ExtensionLoader 去找到要使用的具体的扩展实现，再调用这个扩展实现的同名方法。

扩展实现的选择遵循以下逻辑：

* 读取 `@Adaptive` 注解的 `value` 属性，如果 `value` 没有值则把当前扩展点接口名转换为「点分隔」形式，比如 TroubleMaker 转换为 trouble.maker。然后用这个作为 key 从 URL 上去获取要使用的具体扩展实现。
* 如果上一步没有获取到，则取扩展点接口上的 `@SPI` 注解的 `value` 值作为 key 再去 URL 上获取。

# 怎么启用扩展实现

有些扩展点的扩展实现是可以同时使用多个的，并且可以按照实际需求来启用，比如 Filter 扩展点的众多扩展实现。这就带来两个问题，一个是怎么启用扩展，另一个是扩展是否可以启用。Dubbo 提供了 `@Activate` 注解来标注扩展的启用条件。

```java
public @interface Activate {
  String[] group() default {};
  String[] value() default {};
}
```

众所周知 Dubbo 分为客户端和服务端两侧，`group` 用来指定扩展可以在哪一端启用，取值只能是 `consumer` 或 `provider`，对应的常量位于 CommonConstants。`value` 用来指定扩展实现的开启条件，也就是说如果 URL 上能通过 `getParameter(value)` 方法获取一个不为空（即不为 `false`、`0`、`null` 和 `N/A`）的值，那么这个扩展实现就会被启用。

例如存在一个 Filter 扩展点的扩展实现 FilterX：

```java
@Activate(group = {CommonConstants.PROVIDER}, value = "x")
public class FilterX implements Filter {}
```

如果当前是服务端一侧在加载扩展实现，并且 `url.getParameter("x")` 能拿到一个不为空的值，那 FilterX 这个扩展实现就会被启用。需要注意的是，**@Activate 的 value 属性的值不需要和 SPI 配置文件里的 key 保持一致，并且 value 可以是个数组**。

## 启用扩展实现的方式

第一种启用方式就是上文所讲的让 `value` 作为 url 的 key 并且值不为空，另一种扩展实现的启用就要回到 ExtensionLoader 的 `getActivateExtension(URL url, String key, String group)` 方法。

参数 `key` 表示一个存在于 `url` 上的参数，这个参数的值指定了要启用的扩展实现，多个扩展实现之间用逗号分隔，参数 `group` 表示当前是服务端一侧还是客户端一侧。这个方法把通过参数 `key` 获取到的值拆分后调用了重载方法 `getActivateExtension(URL url, String[] values, String group)`，这个方法就是扩展实现启用的关键点所在。

首先是判断要开启的扩展实现名称列表里有没有 `-default`，这里的 `-` 是减号，是「去掉」的意思，`default` 表示**默认开启的扩展实现**，所以 `-default` 的意思就是要去掉默认开启的扩展实现。所谓**默认开启的扩展实现，其实就是携带了 `@Activate` 注解但是注解的 `value` 没有值的那些扩展实现**，比如 ConsumerContextFilter。以此推论，**如果扩展实现的名称前带了 `-` 就表示这个扩展实现不开启**。

如果没有 `-default` 接着就是迭代 `cachedActivates` 去判断哪些扩展实现是需要使用的，关键方法是 `isActive(String[] keys, URL url)`。这个方法在源码里没有注释，理解起来可能有些困难。实际上就是判断传入的这些 `keys` 是否在 `url` 上存在。

这里有个骚操作，`cachedActivates` 保存的是「扩展实现名称」到「`@Aactivate`」注解的映射，也就是这个 `map` 的 `value` 不是扩展实现的 Class 对象或者实例。因为 `cachedClasses` 和 `cachedInstances` 已经分别保存了两者，只要有扩展实现的名字就可以获取到，没有必要多保存一份。

回到方法的另外一个分支，如果有 `-default`，那就是只开启 `url` 上指定的扩展实现，同时处理一下携带了 `-` 的名称。方法最后把所有要开启的扩展实现放入 `activateExtensions` 集合返回。

## 启用扩展实现的示例

个人认为 Dubbo SPI 这一块适合采用视频的方式进行源码分析，因为这里面有很多逻辑是相互牵连的，依靠文字不太容易讲的明白。所以这里用一个示例来展示上文讲到的扩展实现启用逻辑。假设现在存在以下 5 个自定义 Filter：

```java
public class FilterA implements Filter {}

@Activate(group = {CommonConstants.PROVIDER}, order = 2)
public class FilterB implements Filter {}

@Activate(group = {CommonConstants.CONSUMER}, order = 3)
public class FilterC implements Filter {}

@Activate(group = {CommonConstants.PROVIDER, CommonConstants.CONSUMER}, order = 4)
public class FilterD implements Filter {}

@Activate(group = {CommonConstants.PROVIDER, CommonConstants.CONSUMER}, order = 5, value = "e")
public class FilterE implements Filter {}
```

配置文件 `META-INF/dubbo/internal/org.apache.dubbo.rpc.Filter`

```properties
fa=org.apache.dubbo.rpc.demo.FilterA
fb=org.apache.dubbo.rpc.demo.FilterB
fc=org.apache.dubbo.rpc.demo.FilterC
fd=org.apache.dubbo.rpc.demo.FilterD
fe=org.apache.dubbo.rpc.demo.FilterE
```

首先直接查找消费者端（Consumer）可以使用的 Filter 扩展点的扩展实现：

```java
public static void main(String[] args) {
  ExtensionLoader<Filter> extensionLoader = ExtensionLoader.getExtensionLoader(Filter.class);
  URL url = new URL("", "", 10086);
  List<Filter> activate = extensionLoader.getActivateExtension(url, "", CommonConstants.CONSUMER);
  activate.forEach(a -> System.out.println(a.getClass().getName()));
}
// 输出
// org.apache.dubbo.rpc.filter.ConsumerContextFilter
// org.apache.dubbo.rpc.demo.FilterC
// org.apache.dubbo.rpc.demo.FilterD
```

可以看到自定义扩展实现里的 C 和 D 被启用。A 由于没有 `@Activate` 注解不会默认启用，B 限制了只能在服务端（Provider）启用，E 的 `@Activate` 注解的 `value` 属性限制了 URL 上必须存在名叫 `e` 的参数可以被启用。

接下来添加参数尝试让 E 被启用：

```java
URL url = new URL("", "", 10086).addParameter("e", (String) null);
// 输出
// org.apache.dubbo.rpc.filter.ConsumerContextFilter
// org.apache.dubbo.rpc.demo.FilterC
// org.apache.dubbo.rpc.demo.FilterD
```

可以看到 E 还是没被启用，这是因为虽然 URL 上存在了名为 `e` 的参数，但是值为空，不符合启用规则，这时候只要把值调整为任何不为空（即不为 `false`、`0`、`null` 和 `N/A`）的值就可以启用 E 了。

换另一种方式启用 E：

```java
URL url = new URL("", "", 3).addParameter("filterValue", "fe");
List<Filter> activate = extensionLoader.getActivateExtension(url, "filterValue", CommonConstants.CONSUMER);
// 输出
// org.apache.dubbo.rpc.filter.ConsumerContextFilter
// org.apache.dubbo.rpc.demo.FilterC
// org.apache.dubbo.rpc.demo.FilterD
// org.apache.dubbo.rpc.demo.FilterE
```

添加参数 `filterValue` 并指定值为 `fe`，这里的值要和 SPI 配置文件里的 key 保持一致。调用 `getActivateExtension()` 方法时指定这个参数的名字，这时就可以看到 E 被启用了。

接下来试试去掉默认开启的扩展实现并指定 A 启用：

```java
URL url = new URL("", "", 3).addParameter("filterValue", "fa,-default");
List<Filter> activate = extensionLoader.getActivateExtension(url, "filterValue", CommonConstants.CONSUMER);
// 输出
// org.apache.dubbo.rpc.demo.FilterA
```

加上 `-default` 后 ConsumerContextFilter 和 C 、D 被禁用了，因为他们是默认开启的实现。再回忆一次，默认开启的扩展实现其实就是携带了 `@Activate` 注解但是注解的 `value` 没有值的那些扩展实现。尽管 A 没有携带 `@Activate` 注解，但是这里指定了需要启用，所以 A 被启用。

# 最后

好了，终于分析完了 Dubbo 的这一套 SPI 机制，其实也不算太复杂，只是逻辑绕了一点，有机会我会将本文录制为视频讲解，希望能让大家有更好的理解。
