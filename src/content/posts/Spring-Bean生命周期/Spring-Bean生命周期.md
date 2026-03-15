---
title: Spring Bean 生命周期 — 从懵逼到理清楚
description: "之前面试被问到 Bean 生命周期，只记得\"创建、初始化、销毁\"三板斧，回答得磕磕绊绊。这次认真梳理了一遍源码和资料，发现其实核心脉络没那么复杂，关键是要理解 Spring 在每个阶段都开放了哪些扩展点。"
published: 2025-12-06
tags:
    - Spring
    - Bean
    - 生命周期
category:
    - 后端开发
draft: false
---
> 之前面试被问到 Bean 生命周期，只记得"创建、初始化、销毁"三板斧，回答得磕磕绊绊。这次认真梳理了一遍源码和资料，发现其实核心脉络没那么复杂，关键是要理解 Spring 在每个阶段都开放了哪些扩展点。

## 先上一张全景图

Bean 生命周期简单来说就是四个大阶段：

**实例化 → 属性赋值 → 初始化 → 销毁**

但 Spring 在这四步之间塞了一大堆回调和扩展点，这才是面试考的重点。完整顺序是这样的：

1. **BeanDefinition 解析 & 注册** — 配置文件/注解被解析成 BeanDefinition 对象
2. **BeanFactoryPostProcessor** — 在 Bean 实例化之前，可以修改 BeanDefinition（比如占位符替换）
3. **实例化（Instantiation）** — 调构造方法，创建出对象
4. **属性赋值（Populate）** — 依赖注入，把 @Autowired 那些字段填上值
5. **Aware 接口回调** — BeanNameAware、BeanFactoryAware、ApplicationContextAware 等
6. **BeanPostProcessor#postProcessBeforeInitialization** — 前置处理
7. **初始化回调** — @PostConstruct → InitializingBean#afterPropertiesSet → 自定义 init-method
8. **BeanPostProcessor#postProcessAfterInitialization** — 后置处理（AOP 代理就在这里生成）
9. **Bean 就绪，可以使用了**
10. **销毁回调** — @PreDestroy → DisposableBean#destroy → 自定义 destroy-method

> 我自己记的时候喜欢把它分成三个关键时间点：**实例化前（BeanFactoryPostProcessor）**、**初始化前后（BeanPostProcessor）**、**销毁时**。这样就不容易记混。

## 从源码角度看：doCreateBean 三步走

翻了一下 `AbstractAutowireCapableBeanFactory` 的源码，核心逻辑在 `doCreateBean()` 方法里，其实就三步：

```java
// 简化版，实际源码复杂得多
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, Object[] args) {
    // 第一步：实例化
    BeanWrapper instanceWrapper = createBeanInstance(beanName, mbd, args);

    // 第二步：属性填充（依赖注入）
    populateBean(beanName, mbd, instanceWrapper);

    // 第三步：初始化（Aware 回调 + BeanPostProcessor + init 方法）
    Object exposedObject = initializeBean(beanName, exposedObject, mbd);

    return exposedObject;
}
```

看到这里就挺清晰的了 — 实例化和属性填充完成之后，从 Java 的角度来说这已经是一个"完整的对象"了。但 Spring 还要在 `initializeBean()` 里做一堆增强，这才是框架的价值所在。

## Aware 接口 — 拿到容器的内部资源

Aware 系列接口的套路都一样：实现接口 → Spring 自动回调 → 把资源塞给你。

```java
@Component
public class MyBean implements BeanNameAware, ApplicationContextAware {

    private String beanName;
    private ApplicationContext ctx;

    @Override
    public void setBeanName(String name) {
        // Spring 会把这个 Bean 的名字告诉你
        this.beanName = name;
        System.out.println("我的名字是：" + name);
    }

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        // 拿到整个应用上下文，什么都能干了
        this.ctx = ctx;
    }
}
```

常用的 Aware 接口：

| 接口 | 给你什么 |
|------|---------|
| BeanNameAware | Bean 的名字 |
| BeanFactoryAware | BeanFactory 引用 |
| ApplicationContextAware | ApplicationContext 引用 |
| ResourceLoaderAware | 资源加载器 |
| EnvironmentAware | 环境变量信息 |

> 有个注意点：实现 Aware 接口会让你的代码跟 Spring 框架耦合。如果只是想拿个 Bean，用 @Autowired 就够了，别为了炫技去实现 Aware。

## BeanPostProcessor — 最强扩展点

这个接口我觉得是整个生命周期里最值得深挖的，因为 **AOP 的实现就靠它**。

```java
public interface BeanPostProcessor {
    // 初始化之前调用
    default Object postProcessBeforeInitialization(Object bean, String beanName) {
        return bean;
    }
    // 初始化之后调用（AOP 代理在这里生成）
    default Object postProcessAfterInitialization(Object bean, String beanName) {
        return bean;
    }
}
```

关键点：BeanPostProcessor 对**容器中所有的 Bean** 都生效，不是只针对某一个。Spring 内部也大量使用了这个机制，比如：

- `AutowiredAnnotationBeanPostProcessor` — 处理 @Autowired 注入
- `CommonAnnotationBeanPostProcessor` — 处理 @PostConstruct、@PreDestroy
- `AbstractAutoProxyCreator` — AOP 代理生成

所以面试的时候要是被问到 "AOP 是在 Bean 生命周期的哪个阶段生效的"，答案就是：**在 BeanPostProcessor 的 postProcessAfterInitialization 阶段**。这个约定是有原因的 — before 阶段做配置，after 阶段才做代理，确保代理拿到的是已经配置好的原始类。

## 初始化和销毁的三种方式

Spring 提供了三种方式来自定义初始化和销毁逻辑，执行顺序如下：

**初始化顺序：**
1. `@PostConstruct` 注解方法
2. `InitializingBean#afterPropertiesSet()`
3. 自定义 `init-method`（XML 配置或 @Bean(initMethod="xxx")）

**销毁顺序：**
1. `@PreDestroy` 注解方法
2. `DisposableBean#destroy()`
3. 自定义 `destroy-method`

写个完整的演示：

```java
@Component
public class LifecycleDemo implements InitializingBean, DisposableBean {

    public LifecycleDemo() {
        System.out.println("1. 构造方法 — 实例化");
    }

    @Autowired
    public void setDependency(SomeDependency dep) {
        System.out.println("2. 属性注入");
    }

    @PostConstruct
    public void postConstruct() {
        System.out.println("3. @PostConstruct");
    }

    @Override
    public void afterPropertiesSet() {
        System.out.println("4. InitializingBean#afterPropertiesSet");
    }

    // 假设通过 @Bean(initMethod="customInit") 指定
    public void customInit() {
        System.out.println("5. 自定义 init-method");
    }

    @PreDestroy
    public void preDestroy() {
        System.out.println("6. @PreDestroy");
    }

    @Override
    public void destroy() {
        System.out.println("7. DisposableBean#destroy");
    }

    public void customDestroy() {
        System.out.println("8. 自定义 destroy-method");
    }
}
```

> 日常开发推荐用 `@PostConstruct` / `@PreDestroy`，因为这俩是 JSR-250 标准注解，不绑定 Spring。实现 InitializingBean 接口会耦合框架，一般在框架内部或者需要确保执行顺序的场景才会用。

## 单例 vs 原型 — 生命周期的区别

这个容易忽略：

- **单例（singleton）**：Spring 容器负责完整的生命周期管理，从创建到销毁
- **原型（prototype）**：Spring 只负责创建和初始化，创建完交给你之后就不管了，**销毁回调不会执行**

```java
@Scope("prototype")
@Component
public class PrototypeBean {
    @PreDestroy
    public void cleanup() {
        // 注意：这个方法永远不会被 Spring 调用！
        // 原型 Bean 的资源释放需要自己处理
    }
}
```

这也是为什么原型 Bean 如果持有数据库连接之类的资源时要特别小心 — 你得自己负责释放。

## 踩坑 / 易错点

- **构造方法里不要依赖注入的字段** — 构造方法执行的时候 @Autowired 的属性还没注入呢，拿到的是 null。要用的话放到 @PostConstruct 里。

```java
@Component
public class WrongExample {
    @Autowired
    private UserService userService;

    public WrongExample() {
        // 这里 userService 是 null！别问我怎么知道的...
        userService.doSomething(); // NPE
    }

    @PostConstruct
    public void init() {
        // 这里才能安全使用 userService
        userService.doSomething(); // OK
    }
}
```

- **BeanPostProcessor 影响所有 Bean** — 写 BeanPostProcessor 的时候要注意过滤，别一不小心把不相关的 Bean 也处理了。

- **循环依赖和生命周期** — Spring 通过三级缓存解决循环依赖，实例化后会把早期引用放到三级缓存。但如果是构造器注入的循环依赖，Spring 是解决不了的（因为连实例化都完不成）。

## 小结

Spring Bean 生命周期说白了就是：**创建对象 → 填属性 → 各种回调通知 → 可以用了 → 容器关闭时清理**。Spring 在每个关键节点都留了扩展口，理解了这些扩展点的调用时机，面试就不会再卡壳了。记住 `doCreateBean()` 的三步走（createBeanInstance → populateBean → initializeBean），剩下的都是围绕这三步做的增强。
