---
title: Nest 控制器
categories:
- NestJS
feature_image: "https://picsum.photos/2560/600"
---

### 哲学

目前大量优秀的 Node 库、帮助函数和工具，但最终没有有效地解决好主要问题：架构。Nest 提供一个开箱即用的应用程序架构，允许开发人员和团队创建高度可测试、可扩展、松耦合且易于维护的应用，深受 Angular 的架构影响。

### 概述

控制台负责处理传入的请求和向客户端返回响应。

### 请求对象

在处理器的签名出注入 `@Req()` 装饰器，以便访问请求对象。

```js
@Controller('cat')
export class CatsController {
  @Get()
  findAll(@Req() request: Request): string {
    return 'this action returns all cats';
  }
}
```

以下为 Nest 提供的开箱即用与 `req` 有关的装饰器

| 语法 | 描述 |
| ---  | --- |
| @Request()，@Req() | `req` |
| @Response(), @Res()* | `req` |
| @Nest() | `nest` |
| @Session()	| `req.session` |
| @Param(key?: string)	| `req.params / req.params[key]` |
| @Body(key?: string)	| `req.body / req.body[key]` |
| @Query(key?: string)	| `req.query / req.query[key]` |
| @Headers(name?: string)	| `req.headers / req.headers[name]` |
| @Ip()	| `req.ip` |
| @HostParam() | `req.hosts` |





### 基本实例
```js
@Controller({ host: 'admin.example.com' }) // 子域路由要求传入请求 HTTP 主机匹配特定的值
export class AdminController {
  @Get('ab*cd') // 路由通配符
  @HttpCode(204) // 状态码
  @Redirect('https://docs.nestjs.com', 302) // 重定向
  @Get(':id') // 路由参数
  findAll(@Param('id') id: string， @HostParam('account') account: string) {
    return 'This route uses a wildcard';
  }
}
```

### 请求载荷（Reqeust payloads）
请求的载荷内容通过 `@Body()` 获取。

DTO (Data Transfer Object) 范式是一个对象定义了如何通过网络发送数据。通过 TS 接口或简单的类定义，但推荐使用类。为什么？因类是 JS 标准的一部分，它们在编译后被保留为实际实体。另一方面，由于 TS 接口在转换过程中被删除, Nest 不能在运行时引用它们。这一点很重要，因为诸如管道（Pipe）之类的特性为在运行时访问变量的元类型提供更多的可能性。

```JS
// create-cat.dto.ts
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
// cats.controller.ts
@Post()
async create(@Body() createCatDto: CreateCatDto) {
  return 'This action adds a new cat';
}
```

> `ValidationPipe` 过滤掉不应该被处理器接收的属性。在 `CreateCatDto` 中，白名单分别是 name, age 和 breed 属性







