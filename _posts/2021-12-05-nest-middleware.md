---
title: Nest Middleware
categories:
- NestJS
feature_image: "https://picsum.photos/2560/600"
---

### 概述
中间件是在路由处理器之前调用的函数。它可以访问 `request` 和 `response` 对象，以及应用程序请求-响应周期中的 `next()` 中间件函数。

Nest 中间件默认等同于 Express 中间件。以下为执行的任务：
* 执行任何代码
* 对请求和响应对象更改
* 结束请求-响应周期
* 调用堆栈中的下一个中间件函数
* 如果当前中间函数没有终止请求-响应周期，它必须调用 `next()` 传递给下一个中间件函数，否则，请求将被挂起

自定义中间件可以是一个函数或是具有 `@Injectable` 装饰器的函数，应实现 `NestMiddleware` 接口。

```js
@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    console.log('Request...');
    next();
  }
}
```

### 依赖注入
中间件如同 providers 和 controllers 一样，通过 `constructor` 注入依赖。

### 应用中间件
中间件不能在 `@Module({})` 装饰器中使用，必须使用模块类的 `configure()` 方法来设置它们。包含中间件的模块必须实现 `NestModule` 接口。
```js
@Module({
  imports: [CatsModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware) // loggerMiddleware 绑定特定的路由
      .forRoutes('cats');
      // .forRoutes({ path: 'cats', method: RequestMethod.GET })
      // forRoutes({ path: 'ab*cd', method: RequestMethod.ALL });
      // .forRoutes(CatsController);

      // .exclude(
      //   { path: 'cats', method: RequestMethod.GET },
      //   { path: 'cats', method: RequestMethod.POST },
      //   'cats/(.*)',
      // )
  }
}
```

### 函数式中间件
函数式中间件没有成员，方法及依赖关系的函数
```js
export function logger(req: Request, res: Response, next: NextFunction) {
  console.log('Request...');
  next();
}

consumer.apply(logger).forRoutes(CatsController);
```
> 当中间件没有任何依赖关系时，应考虑使用函数式中间件

### 多个中间件
为了绑定顺序执行的多个中间件，在 `apply()` 方法内用逗号分隔它们。
```js
consumer.apply(cors(), helmet(), logger).forRoutes(CatsController);
```

### 全局中间件
```js
const app = await NestFactory.create(AppModule);
app.use(logger);
```