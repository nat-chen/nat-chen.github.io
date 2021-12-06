---
title: Nest Exception Filters
categories:
- NestJS
feature_image: "https://picsum.photos/2560/600"
---

### 概述
Nest 内置了异常层用于处理整个应用程序中所有抛出的异常。当异常未被应用的代码捕获时，它将捕获并返回合适的响应。

开箱即用，此操作由内置的全局异常过滤器执行，它处理的异常类型为 `HttpException`（和它的子类）。当一个异常未被识别（既不是 `HttpException` 或是继承自 `HttpException` 的类），内置的异常过滤器生成如下默认 JSON 响应：
```js
{
  "statusCode": 500,
  "message": "Internal server error"
}
```

> 全局异常过滤器部分支持 `http-errors` 库，任何抛出的异常包含 `statusCode` 和 `mesage` 属性将被适当的返回响应（而非默认未被识别的异常 `InternalServerErrorException`）

### 抛出标准异常
`HttpExcepiton` 构造器包含两个必填的参数：
* `response`: 定义 JSON 响应体，可以是 `string` 或是 `object` 类型
* `status`: 定义 HTTP 状态码

```js
@Get()
async findAll() {
  throw new HttpException({
    status: HttpStatus.FORBIDDEN,
    error: 'This is a custom message',
  }, HttpStatus.FORBIDDEN);
}
/*
{
  "status": 403,
  "error": "This is a custom message"
}
*/
```

### 自定义异常
```js
export class ForbiddenException extends HttpException {
  constructor() {
    super('Forbidden', HttpStatus.FORBIDDEN);
  }
}

@Get()
async findAll() {
  throw new ForbiddenException();
}
```

### 内置 HTTP 异常
* `BadRequestException`
* `UnauthorizedException`
* `NotFoundException`
* `ForbiddenException`
* `NotAcceptableException`
* `RequestTimeoutException`
* `ConflictException`
* `GoneException`
* `HttpVersionNotSupportedException`
* `PayloadTooLargeException`
* `UnsupportedMediaTypeException`
* `UnprocessableEntityException`
* `InternalServerErrorException`
* `NotImplementedException`
* `ImATeapotException`
* `MethodNotAllowedException`
* `BadGatewayException`
* `ServiceUnavailableException`
* `GatewayTimeoutException`
* `PreconditionFailedException`

### 异常过滤器
异常过滤器可以控制精确的控制流和返回给客户端的响应内容。所有的异常过滤器应实现泛型 `ExceptionFilter<T>` 接口。`@Catch(HttpException)` 装饰器绑定所有的元数据到异常过滤器上，告知 Nest 这个特定的过滤器正在 `HttpException` 类型的异常。它支持传多个参数，用于同时设置多个异常类型。

```js
@Catch(HttpException)
export class HttpExcepionFilter implements ExceptionFilter {
  catch(excpetion: HttpException, host: ArgumentsHost) {
    const ctx = host.switchHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    const status = exception.getStatus();

    response
      .status(status)
      .json({
        statusCode: status,
        timestamp: new Date().toISOString(),
        path: request.url,
      })
  }
}
```

### 参数主机
`catch()` 参数 `exception` 是当前正在处理的异常对象。该 `host` 参数是一个 `ArgumentsHost` 对象，访问应用中的上下文获取 `Request` 和 `Response` 对象。

### 绑定过滤器
尽可能使用类而非实例。Nest 在整个模块中重复使用一个类的实例，减少了内存使用。另外将创建实例的交给框架及开启了依赖注入。

```js
@Post()
@UseFilters(HttpExceptionFilter) // 类
// @UseFilters(new HttpExceptionFilter()) 实例
async create(@Body() createCatDto: CreateCatDto) {
  throw new ForbiddenException();
}

// 创建全局的过滤器
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalFilter(new HttpExceptionFilter());
}
```

全局过滤器可以在整个应用中使用，针对每个控制器和路由处理器。在依赖注入方面，全局过滤器在任何模块的外部注册不能注入依赖。为了解决问题，参照如下代码
```js
// app.module.ts
@Module({
  providers: [
    { provide: APP_FILTER },
    useClass: HttpExceptionFilter,
  ]
});
```

### 捕获所有
为了捕获所有未被处理的异常，不传参给 `@Catch()` 装饰器
```js
import {
  ExceptionFiler,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
} from '@nestjs/common';
import { HttpAdapterHost } from '@nestjs/core';

@Catch()
export class AllExceptionFilter implements ExceptionFilter {
  constructor(private readonly httpAdapterHost: HttpAdapterHost) {}
  catch (exception: unknow, host: ArgumentsHost): void {
    // In certain situations `httpAdapter` might not be available in the
    // constructor method, thus we should resolve it here.
    const { httpAdapter } = this.httpAdapterHost;
    const ctx = host.switchToHttp();
    const httpStatus =
      exception instanceof HttpException
        ? exception.getStatus()
        : HttpStatus.INTERNAL_SERVER_ERROR;
    const responseBody = {
      statusCode: httpStatus,
      timestamp: new Date().toISOString(),
      path: httpAdapter.getRequestUrl(ctx.getRequest()),
    };
    httpAdapter.reply(ctx.getResponse(), responseBody, httpStatus);
  }
}
```

### 继承
```js
import { Catch, ArgumentsHost } from '@nestjs/common';
import { BaseExceptionFilter } from '@nestjs/core';

@Catch()
export class AllExceptionFilter extends BaseExceptionFilter {
  catch(exception: unknow, host: ArgumentsHost) {
    super.catch(exception, host);
  }
}
```
> 方法和控制器范围的过滤器继承 `BaseExceptionFilter` 不应使用 `new` 实例化，而应该让框架自动实例化。









