---
title: Nest Custom Decorator
categories:
- NestJS
feature_image: "https://picsum.photos/2560/600"
---

### 参数装饰器
```js
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const User = createParamDecorator(
  (data: unknown, ctx: ExcecutionContext) => {
    const requet = ctx.switchToHttp().getRequest();
    return request.user;
  }
)

@Get()
async findOne(@User() user: UserEntity) { // user 等同于 request.user
  console.log(user);
}
```

### 与管道结合
```js
@Get()
async findOne(
  @User(new ValidationPipe({ validateCustomDecorators: true }))
  user: UserEntity,
) {}
```
> 注意 `validateCustomDecoratos` 选项必须设置为 true。默认情况下，`ValidationPipe` 不验证使用自定义装饰器注释的参数

### 装饰器合成
Nest 提供一个 helper 方法用于合成多个装饰器。
```js
import { applyDecorators } from '@nestjs/common';
export function Auth(...roles: Role[]) {
  return applyDecorators(
    SetMetadata('roles', roles),
    UseGuards(AuthGuard, RolesGuard),
    ApiBearerAuth(),
    ApiUnauthorizedResponse({ description: 'Unauthorized' }),
  );
}

@Get('user')
@Auth('admin')
findAllUsers() {}
```