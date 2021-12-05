---
title: Nest 守卫、鉴权及授权
categories:
- NestJS
feature_image: "https://picsum.photos/2560/600?image=872"
---

### 守卫 （Guards）
守卫是一个类，标注了 `@Injectable` 装饰器，且实现 `CanActivate` 接口，这个接口返回当前请求是否应被允许的布尔值 （同步或异步 `Promise` `Observable`），返回值用于判断下一步操作应该如何执行，为 `true` 请求继续，为 `false` 请求被拒绝。

守卫抛出的任何异常都会被异常过滤器处理，返回 `fasle` 将默认抛出 `ForbiddenException`, 可指定抛出其他错误（如 `throw new UnauthorizedException()`）。

它的执行时机是客户端发起一个 HTTP 请求到路由处理器之间，决定一个请求是否应被路由处理器接收，通常也被称为鉴权（含授权）。要注意是守卫执行时机在中间件（`middleware`）之后，和拦截器（`interceptor`）及管道（`pipe`）之前。

在 Express 应用中通常是中间件处理，但存有个问题是调用了 `next()` 后不知道接下来哪个处理器执行，而守卫可以访问到 `ExecutionContext` 实例，便知道下一步要执行的逻辑。

#### 授权守卫
授权守卫的一个使用场景是：特定的路由只能被已鉴权的用户访问（比如请求的头部含 token）

```js
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    const request = context.switchToHttp().getRequest();
    return customValidateRequest(request); // 自定义的验证请求的函数
  }
}
```

#### 执行上下文 （Execution context）
`ExecutionContext` 继承自 `ArgumentsHost`，除此之外，添加了若干个 helper 方法用于提供关于当前执行进程的额外信息。
通过 `getRequest()` 方法可以访问到当前的请求对象。


#### 基于角色的授权（Role-based authentication）
```js
@Injectable()
export class RolesGuard implements CanActivate {
  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    return true;
  }
}
```

#### 绑定守卫（Binding guards）
与管道 `pipes` 、异常过滤器 `exception filters` 一样，守卫的使用范围可以是控制器、方法或全局。
```js
@Controller('cats')
@UseGuards(RolesGuard)
export class CatController {}
```
以上的 `@UseGuards(RolesGuard)`，传一个类意味着由框架实例化，并启用依赖注入。另外一种形式是传入实例 `@UseGuards(new RolesGuard())`。

全局守卫 `useGlobalGuards` 应用于整个应用。在依赖注入方面，由于不属于任何一个模块，不能执行依赖项插入，解决办法如下
```js
import { APP_GUARD } from '@nestjs/core';
@Module({
  providers: [
    {
      provide: APP_GUARD,
      useClass: RolesGuard,
    }
  ]
});
export class AppModule {}
```

#### 单个处理器设置角色 （Setting roles per handler）
Nest 通过 `@SetMetadata()` 装饰器给路由处理器提供用户自定义的元数据（custom metadata）。
```js
// roles.decorator.ts
export const Roles = (...roles: string[]) => SetMetadata('roles', roles);

// cats.controller.ts
@Post()
@Roles('admin') // roles 此处为 ['admin']
async create(@Body() createCatDto: CreateCatDto) {
  this.catService.create(createCatDto);
}
```

#### 与 `RolesGuard` 结合的例子
```js
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(prviate reflector: Reflector) {} // reflector helper 访问路由的角色信息
  canActivate(context: ExecutionContext): boolean {
    const roles = this.reflector.get<string[]>('roles', context.getHandler());
    if (!roles) {
      return true;
    }
    const request = context.swithToHttp().getRequest();
    const user = request.user; // 假使 request 请求对象含有 user 实例及角色信息（实战建议在 request 添加已授权的 user 对象）
    return matchRoles(roles, user.roles);
  }
}
```

### 认证（Authentication）

Nest 集成了 **Passport** 库，内置了 `@nestjs/passport` 模块。Passport 执行了一下步骤
* 验证用户的凭证（比如用户/密码，JWT 或是身份验证令牌 identity token）
* 管理已认证的状态（比如 JWT，或是创建一个 Express 会话）
* `Request` 对象添加已认证的 user 信息，以便路由处理器访问。

#### 认证要求
1、首先验证用户的用户名及密码，一旦通过后服务端在响应的头部签发 JWT 作为 bearer token 作为后续接口的验证。
2、创建一个受保护的路由仅限被含有效的 JWT 访问。

Passport 提供一个 **passport-local** 策略，实现基于用户名/密码的认证机制。

#### Passport 策略
使用原生 Passport, 配置策略时需提供两个信息：
1. 指定一些配置项信息（如在 JWT 策略中，应提供 secret 对 token 签名）
2. 验证回调（verify callback）。在验证成功时返回完整的用户信息，失败时返回 null (如密码不匹配或是用户未找到场景)。

在实际项目中，应该使用像加盐单向的哈希算法的 **bcrypt** 之类的库。使用这种方法，只储存已散列的密码。通过已储存的密码与输入的密码比对。

#### 实现 Passport local
passport-local 默认 `request` 对象带有 `username` 和 `passport` 属性。传一个配置对象用于指定不同的属性名（像 `super({usernameField: 'email' })`）。

对于每个策略，Passport 使用适当的特定于策略的一组参数调用 verify 函数(使用 `@nestjs/Passport` 中的 `validate()` 方法实现)。对于本地策略，Passport 需要一个具有以下签名的 `validate()` 方法: `validate(username: string, password: string): any`。

通常，每种策略的 validate() 方法的惟一显著差异是如何确定用户是否存在和是否有效。

```js
// local.strategy.ts
import { Strategy } from 'passport-local';
export class LocalStrategy extends PassportStrategy(Strategy) {
  constructor(private authService: AuthService) {
    super();
  }
  async validate(username: string, password: string): Promise<any> {
    const user = await this.authService.customValidateUser(username, password);
    if (!user) {
      throw new UnauthorizedException();
    }
    return user;
  }
}
```

#### 内置 Passport 守卫
守卫主要功能是决定请求是否由路由处理器处理，同时能调用 `Passport` 策略自动处理类似于检索凭证、运行 verify 函数、创建 user 属性。

#### 登录路由
路由有两种使用访问场景：
* 带有凭证的用户方可访问的路由
* 登录路由

Passport 基于从 `validate()` 方法返回的值，自动创建一个 `user` 属性赋值给 `req` 请求对象。

```js
import { Controller, Request, Post, UseGuards } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

// 版本 1
@Controller()
export class AppController {
  @UseGuards(AuthGuard('local'))
  @Post('auth/login')
  async login(@Request() req) {
    return req.user;
  }
}

// 版本 2，推荐
@UseGuards(LocalAuthGuard)
@Post('auth/login')
async login(@Request() req) {
  return req.user;
}

@Injectable()
export class LocalAuthGuard extends AuthGuard('local') {}
```

Passport 基于从 `validate()` 方法返回的值，自动创建一个 `user` 属性赋值给 `req` 请求对象。

#### JWT 功能
当用户使用用户名/密码验证身份后返回有效的 JWT（bearer token）作为后续其他受访问保护的 API。`passport-jwt` 实现了 `JWT` 策略的 Passport 包。

```js
// auth/auth.service.ts
import { JwtService } from '@nestjs/jwt';

@Injectable()
export class AuthService {
  constructor(
    private usersService: UsersService,
    private jwtService: JwtService
  ) {}

  async validateUser(username: string, pass: string): Promise<any> {
    const user = await this.usersService.findOne(username);
    if (user && user.password === pass) {
      const { password, ...result } = user;
      return result;
    }
    return null;
  }

  async login(user: any) {
    // `userId` 赋值给 `sub` 属性是为了保持与 JWT 标准一致。
    const payload = { username: user.username, sub: user.userId };
    return {
      access_token: this.jwtService.sign(payload), //生成 token
    }
  }
}

// auth.module.ts
@Module({
  imports: [
    UsersModule, // 自定义
    PassportModule, // @nestjs/passport
    JwtModule.register({
      secret: jwtConstants.secret,
      signOptions: { expiresIn: '60s' }
    }),
  ],
  providers: [AuthService, LocalStrategy], // LocalStrategy 为自定义本地策略
  exports: [AuthService],
})

// app.controller.ts
@Controller()
export class AppController {
  constructor(private authService: AuthService) {}

  @UseGuards(LocalAuthGuard) // 守卫用于验证用户名和密码
  @Post('auth/login')
  async login(@Request() req) {
    return this.authService.login(req.user); // AuthService 的 login 函数执行登录
  }
}
```

#### 实现 Passport JWT
```js
// jwt.strategy.ts
import { ExtractJwt, Strategy } from 'passport-jwt';
import { PassportStrategy } from '@nestjs/passport';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor() {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(), //获取 JWT 从 `Request` 对象
      ignoreExpiration: false, // 将 JWT 是否过期的责任委托给 Passport 模块。若过期则请求被拒绝且返回 `401 Unauthorized`
      secretOrKey: jwtConstants.secret, // 提供对称的秘密来签署令牌
    })
  }

  async validate(payload: any) {
    return { userId: payload.sub, username: payload.username }; // 返回值作为构建 req.user
  }
}

// auth.module.ts
@Module({
  imports: [
    UsersModule,
    PassportModule, // @nestjs/passport
    JwtModule.register({ // @nestjs/passport
      secret: jwtConstants.secret,
      signOptions: { expiresIn: '60s' },
    }),
  ],
  providers: [AuthService, LocalStrategy, JwtStrategy],
  exports: [AuthService],
});
export class AuthModule {}

// 最终，使用 JwtAuthGuard 去继承内置的 AuthGuard
@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {}

```

通过导入 JWT 签名时使用的相同密钥，确保 Passport 执行的验证阶段和 `AuthService` 执行的签名阶段使用公共密钥。

#### 实现保护的路由和 JWT 策略守卫
结合以上，我们可以实现受保护的路由及相关的守卫

```js
@Controller()
export class AppController {
  constructor(private authService: AuthService) {}

  @UseGuards(LocalAuthGuard)
  @Post('auth/login')
  async login(@Request() req) {
    return this.authService.login(req.user);
  }

  @UseGuards(JwtAuthGuard)
  @Get('profile')
  getProfile(@Request() req) {
    return req.user;
  }
}
```

#### 继承 Guards
继承 AuthGuard 实现覆写错误处理及认证的逻辑

```js
@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  canActivate(context: ExecutionContext) {
    // Add your custom authentication logic here
    // for example, call super.logIn(request) to establish a session.
    return super.canActivate(context);
  }

  handleRequest(err, user, info) {
    // You can throw an exception based on either "info" or "err" arguments
    if (err || !user) {
      throw err || new UnauthorizedException();
    }
    return user;
  }
}
```

#### 全局开启认证
```js
providers: [
  {
    provide: APP_GUARD,
    useClass: JwtAuthGuard,
  },
],
```

#### 请求范围的策略
```js
// local.strategy.ts
constructor(private moduleRef: ModuleRef) { // @nestjs/core
  super({
    passReqToCallback: true, // 请求对象获取当前上下文对象
  });
}

async validate(
  request: Request,
  username: string,
  password: string,
) {
  const contextId = ContextIdFactory.getByRequest(request); // 创建 contextId 基于请求对象
  // "AuthService" is a request-scoped provider
  const authService = await this.moduleRef.resolve(AuthService, contextId); // resolve 异步返回一个 request-scoped 的 AuthService 实例
  ...
}
```

#### 自定义 Passport
```js
PassportModule.register({ session: true }); // 将选项对象直接传递给Passport实例

constructor(private authService: AuthService) {
  super({
    usernameField: 'email',
    passwordField: 'password',
  });
}
```

#### 命名策略
```js
export class JwtStrategy extends PassportStrategy(Strategy, 'myjwt')
```

### 权限（Authorization）
权限用于决定一个用户可以做什么的过程。权限和认证相互独立，但权限依赖认证机制。

#### 基本 RBAC 实现
RBAC (role-based access control) 是一个基于角色和权限等级的中立的访问控制策略。

```js
// role.enum.ts
export enum Role {
  User = 'user',
  Admin = 'admin',
}

// roles.decorator.ts
export const ROLES_KEY = 'roles';
export const Roles = (...roles: Role[]) => SetMetadata(ROLES_KEY, roles);

// cats.controller.ts
@Post()
@Roles(Role.Admin)
create(@Body() createCatDto: CreateCatDto) {
  this.catService.create(createCatDto);
}

// roles.guard.ts
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<Role[]>(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (!requiredRoles) {
      return true;
    }
    const { user } = context.switchToHttp().getRequest();
    return requiredRoles.some((role) => user.roles?.includes(role));
  }
}
```

#### 基于权利（Claims）的权限
当一个身份创建后被信任一方指派了一个或多个权利。一个权利是一个键值对用于表示能做什么及不能做什么。

要在 Nest 中实现基于权利的权限，参考 RBAC 部分的步骤，仅仅有一个显著区别：比较许可(permissions)而不是角色。

```js
// cats.controller.ts
@Post()
@RequirePermissions(Permission.CREATE_CAT) // Permission 枚举对象包含所有的许可（类似 role）
create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

#### 集成 CASL
CASL 是一个权限库用于限制用户可以访问哪些资源。它被设计为可渐进式增长的，从基础权利权限到完整的基于主题和属性的权限都可以实现。

例子的需求如下：
1. 管理员可以管理所有实体（CRUD 所有操作）
2. 用户对所有内容有阅读权限
3. 用户可以更新自己的文章（`article.authorId === userId`）
4. 已发布的不能被删除（`article.isPublished === true`）

```js
export enum Action {
  Manage = 'manage', // `manage` 在 CASL 是一个特殊的关键字表示任何操作
  Create = 'create',
  Read = 'read',
  Update = 'update',
  Delete = 'delete'
}

// User and Article entity
class User {
  id: number;
  isAdmin: boolean;
}

class Article {
  id: number;
  isPublished: boolean;
  authorId: number;
}


// all 在 CASL 中是一个特殊的关键字表示任何对象。
type Subjects = InferSubjects<typeof Article | typeof User> | 'all';
export type AppAbility = Ability<[Action, Subjects]>;

@Injectable()
export class CaslAbilityFactory {
  createForUser(user: User) {
    const { can, cannot, build } = new AbilityBuilder<Ability<[Action, Subjects]>>(Ability as AbilityClass<AppAbility>);
    if (user.isAdmin) {
      can(Action.Manage, 'all'); // read-write access to everything
    } else {
      can(Action.Read, 'all'); // read-only access to everything
    }
    can(Action.Update, Article, { authorId: user.id });
    cannot(Action.Delete, Article, { isPublished: true });
    return build({
      // detectSubjectType 选项用于告知 CASL 如何从对象中获取键名。
      detectSubjectType: item => item.constructor as ExtractSubjectType<Subjects>
    })
  }
}

@Module({
  providers: [CaslAbilityFactory],
  exports: [CaslAbilityFactory],
});

// 在 constructor 注入 CaslAbilityFactory 对象
const user = new User();
user.isAdmin = false;

const ability = this.caslAbilityFactory.createForUser(user);
ability.can(Action.Read, Article); // true
ability.can(Action.Delete, Article); // false
ability.can(Action.Create, Article); // false
```






