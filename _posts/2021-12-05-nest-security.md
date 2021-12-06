---
title: Nest Security
categories:
- NestJS
feature_image: "https://picsum.photos/2560/600"
---

### 加密和哈希
加密是编码信息的过程。这个过程转换原始信息（明文）到另外一种密文的形式。理想情况下，只有授权放可以将密文转换成明文。加密是一个双向的函数，加密可以被合适的 key 解密。

哈希是通过给定的 key 转换成另外一个值的过程。散列函数基于数学算法创建一个新值。一旦哈希完成，是无法从输出值计算回输入值。

#### 加密
Node 提供了内置的 crypto 模块用于加密和解密字符串、数字、buffers 或流等。
```js
// AES 高级加密系统 (Advanced Encryption System) aes-256-ctr，CTR 加密模式
import { createCipheriv, randomBytes, scrypt } from 'crypto';
import { promisify } from 'util';

const iv = randomBytes(16);
const password = 'Password used to generated key';

// The key length is dependent on the algorithm.
// In this case for aes256, it is 32 bytes.

const key = (await promisify(scrypt)(passport, 'salt', 32)) as Buffer;
const cipher = createCipheriv('aes-256-ctr', key, iv);
const textToEncrypt = 'Nest';
const encryptedText = Buffer.concat([
  cipher.update(textToEncrypt),
  cipher.final()
])

// 解密 encryptedText
import { createDecipheriv } from 'crypto';
const decipher = createDecipheriv('aes-256-ctr', key, iv);
const decryptedText = Buffer.concat([
  decipher.update(encryptedText),
  decipher.final()
]);
```

### 哈希
推荐使用 bcrypt 或 argon2 哈希库。哈希算法主要有MD4、MD5、SHA。
```js
import * as bcrypt from 'bcrypt';

const salt = await bcrypt.genSalt();
const password = 'random_password';
const hash = await bcrypt.hash(password, salt)

// 比对密码与哈希的密码
const isMatch = await bcrypt.compare(passport, hash);
```

盐 (Salt) 在密码学中，是指通过在密码任意固定位置插入特定的字符串，让散列后的结果和使用原始密码的散列结果不相符，这种过程称之为 "加盐"。密码不能以明文形式保存到数据库中，否则数据泄露密码就会被知道。而一般的加密方式由于加密规则固定，很容易被破解，安全系数不高（MD5 现已能够破解，不再足够安全。即使破解不了，黑客拿到数据库后，用常用的密码数据集进行匹配，也能拿到用户的密码）。密码加盐的加密方式，能很好的解决这一点。
密码加盐里包含随机值和加密方式。随机值是随机产生的，并且以随机的方式混在原始密码里面，然后按照加密方式生成一串字符串保存在服务器。换言之，这个是单向的，电脑也不知道客户的原始密码，即使知道加密方式，反向推出的加密前的字符串也是真正密码与随机值混合后的结果，从而无法解析用户的真正密码。那么是如何验证密码的呢？当你再次输入密码，会以相同的加盐方式生成字符串（盐要提前记录下来，以便后续密码验证），如果和之前的一致，则通过。而其它用户无法获得这种加密方式：即生成哪些随机数，以什么方式混入进去。

### Helmet
通过适当地设置 HTTP 头，Helmet 可以帮助保护应用免受一些众所周知的 Web 漏洞的影响。通常，Helmet 只是14个较小的中间件函数的集合，它们设置与安全相关的 HTTP 头。

### CORS
跨域资源共享（CORS）允许从其他域请求资源的机制。在底层，Nest 使用了 Express cors 包。

```js
// REST 版本 1
const app = await NestFactory.create(AppModule);
app.enableCors();
// REST 版本 2
const app = await NestFactory.create(AppModule, { cors: true });

await app.listen(3000);
```

### CSRF 保护
跨站点请求伪造（CSRF or XSRF）是一种恶意利用网站，其中未经授权的命令从 Web 应用程序信任的用户传输。要减轻此类攻击，使用 csurf 软件包。
```js
import * as csurf from 'csurf';
app.use(csurf());
```

### 限速
为了保护应用程序免受暴力攻击，通常采取是速率限制。
```js
@Module({
  imports: [
    ThrottlerModule.forRoot({
      ttl: 60, // the time to live
      limit: 10, // maximum number of requests within the ttl
    }),
  ],
})
export class AppModule {}
```