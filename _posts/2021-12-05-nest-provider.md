---
title: Nest Provider
categories:
- NestJS
feature_image: "https://picsum.photos/2560/600"
---

### 概述
许多基本 Nest 类都可以视为提供者（`provider`），像 `services`, `repositories`, `factories` 或 `helpers`。它们都可以通过 `constructor` **注入依赖**关系。这意味着对象间可以创建各种关系，并且 “连接” 对象实例的功能在很大程度上可以委托给 Nest 运行时系统。

控制器用于处理 HTTP 请求和委派更复杂的任务给提供者，提供者只是一个普通的类，用 `@Injectable()` 装饰器注释，在模块中声明为 `providers`。


### 服务（Services）
`@Injectable()` 装饰器附带元数据，且声明这个类可以被 Nest IoC 容器管理。

### 依赖注入（Dependency injection）
Nest 是建立在强大的设计模式, 通常称为依赖注入。

依赖是服务或对象用于给类提供若干功能。依赖注入是一种设计模式用于类从外部资源获取而非自己创建依赖。（摘抄自 Angular 文档）

### 生命周期（Scopes）
Provider 通常具有与应用程序生命周期同步的生命周期（scope 作用域）。在启动应用程序时，必须解析每个依赖项，因此必须实例化每个提供程序。同样，当应用程序关闭时，每个 provider 都将被销毁。但是，有一些方法可以改变 provider 生命周期的请求范围（request-scoped）。

### 自定义 providers
Nest 有一个内置的**控制反转**（"IoC"）容器，可以解决 providers 之间的关系。 此功能是上述依赖注入功能的基础。

### 可选的 providers
依赖有时可能不需要被解析，如类依赖一些配置对象，如果没有传递，则应使用默认的值。在这种场景下，依赖变成可选，应使用 `@Optional()` 装饰器。

```js
@Injectable()
export class HttpService<T> {
  constructor(@Optional() @Inject('HTTP_OPTIONS') private httpClient: T) {}
}
```

### 基于属性的注入
以上使用均基于构造函数的注入，即通过 constructor 注入 providers。Nest 支持基于属性的注入。

```js
@Injectable()
export class HttpService<T> {
  @Inject('HTTP_OPTIONS')
  private readonly httpClient: T;
}
```