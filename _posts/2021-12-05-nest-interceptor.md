---
title: Nest Interceptor
categories:
- NestJS
feature_image: "https://picsum.photos/2560/600"
---

### 概述
拦截器是具有注解为 `@Injectable()` 的类, 应该实现了 `NestInterceptor` 接口。拦截器可以拦截在请求之后路由接收之前，或是路由响应之后。

拦截器受 AOP (切面编程，Aspect Oriented Programming) 影响，支持如下功能：
* 在函数执行之前/之后绑定额外的逻辑
* 转换从函数返回的结果
* 转换从函数抛出的异常
* 扩展基本函数行为
* 根据特定的条件完全重写函数 (例如, 缓存目的)

### 基础
每个拦截器实现 `intercept()` 方法，它接收两个参数，第一个是 `ExecutionContext` 实例，第二个参数为 `CallHandler`。

### 执行上下文
继承了 `ArgumentHost`, `ExecutionContext` 添加若干新的 helper 方法提供关于当前执行上下文进程的更多的信息。

### Call handler
`CallHandler` 接口实现了 `handle()` 方法，以便在拦截器中调用路由处理器方法。如果不在 `intercept()` 中调用 `handle()`，路由处理器方法将不会执行。

这种方式意味着 `interceptor()` 高效地包装了请求和响应流。因此在最终的路由处理器执行前或后可以实现自定义逻辑。我们非常清楚在 `intercept()` 方法的 `handle()` 之前执行一些自定义的代码，但如何影响后续？`handle()` 返回一个 `Observable` 对象，我们可以使用强大的 RxJS 去操作响应。使用面向切面编程，路由处理器的调用（如 `handle()`）被称之为切入点（PointCut），表示在某个时间点上插入额外的逻辑。

### 截取切面
```js
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  interceptor(context: ExecutionContext, next: CallHandler): Observable<any> {
    console.log('Before...')；
    const now = Date.now();
    return next.handle().pipe( // handle() 返回 RxJS Observable
      tap(() => console.log(`After... ${Date.now() - now}ms`)),
    )
  }
}

// 绑定拦截器
@UseInterceptor(LogginInterceptor) // 类
// @UseInterceptors(new LoggingInterceptor()) 实例
export class CatController {}

app.useGlobalInterceptors(new LoggingInterceptor()) // 全局过滤器

@Module({
  providers: [
    {
      provide: APP_INTERCEPTOR,
      useClass: LoggingInterceptor,
    }
  ]
})
```

### 响应映射
`handle()` 返回一个 `Observable` 对象。这个流包含了从路由处理器返回的值，可通过 RxJS 中的 `map()` 修改值。
```js
// transform.interceptor.ts
import { Obserable } from 'rxjs';
import { map } from 'rxjs/operators';

export interface Response<T> {
  data: T;
}

@Injectable()
export class TransformInterceptor<T> implements NestInterceptor<T, Response<T>> {
  intercept(context: ExecutioinContext, next: CallHandler): Observable<Response<T>> {
    return next.handle().pipe(map(data => ({ data })));
  }
}
```

### 异常映射
RxJs 的 `catchError()` 覆写抛出的异常
```js
import { Observable, throwError } from 'rxjs';
import { catchError } from 'rxjs/operators';

@Injectable()
export class ErrorsInterceptor implements NestInterceptor {
  interceptor(context: ExceptionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(catchError(err => throwError(new BadGatewayException())))
  }
}
```

### 流覆写
```js
import { Observable, of } from 'rxjs';

@Injectable()
export class CacheInterceptor implements NestInterceptor {
  interceptor(context: ExecutionContext, next: CallHandler): Observable<any> {
    const isCached = true;
    if (isCached) {
      return of ([]);
    }
    return next.handle();
  }
}
```

### 更多操作
```js
// 处理超时的请求，若一段时间不响应，直接返回一个错误的响应
@Injectable()
export class TimeoutInterceptor implemens NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      timeout(5000),
      catchError(err => {
        catchError(err => {
          if (err instanceof TimeoutError) {
            return throwError(new RequestTimeoutException());
          }
          return throwError(err);
        })
      })
    )
  }
}
```



