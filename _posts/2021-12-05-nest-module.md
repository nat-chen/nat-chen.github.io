---
title: Nest Module
categories:
- NestJS
feature_image: "https://picsum.photos/2560/600"
---

### 概述
模块是具有 `@Module()` 装饰器的类。`@Module()` 装饰器提供了元数据，可供 Nest 用于组织应用程序架构。

每个应用最少有个根模块。根模块用于 Nest 构建应用图（application graph）的起点。它包含内部的数据结构用于 Nest 解析模块和提供者间的关系和依赖。

`@Module()` 装饰器有如下属性
* `providers`： 由 Nest 注入器实例化 provider，在当前模块中共享
* `controllers`: 控制器
* `imports`：导入模块的列表
* `exports`: 将本模块的 provider 导出到其他模块共享功能

### 共享模块
在 Nest 中，默认情况下，模块是单例，可以轻松地在多个模块之间共享同一个提供者实例。
实际上，每个模块都是一个共享模块，一旦创建实例就能被任意模块重复使用。需将 `provider`（如 `CatService`） 添加到模块 exports 数组中。

### 全局模块
`@Global()` 装饰器使模块全局化。全局模块只能被注册一次，最好在根或核心模块。
```js
@Global()
@Module({
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService]
});
```

### 动态模块
创建自定义模块，可动态注册和配置程序。动态模块返回的属性扩展而非覆盖 `@Module` 装饰器中定义的基本模块的元数据。

```js
import { Module, DynamicModule } from '@nestjs/common';
import { createDatabaseProviders } from './database.providers';
import { Connection } from './connetion.provider';

@Module({
  providers: [Connection],
});

export class DatabaseModule {
  static forRoot(entities = [], options?): DynamicModule {
    const providers = createDatabaseProviders(options, entities);
    return {
      global: true, // 全局范围注册动态模块
      module: DatabaseModule,
      providers: providers,
      exports: providers,
    };
  }
}

// 导入及配置 DatabaseModule
@Module({
  imports: [DatabaseModule.forRoot([User])]
});
export class AppModule {}

// re-export 动态模块
@Module({
  imports: [DatabaseModule.forRoot([User])],
  exports: [DatabaseModule],
})
```




