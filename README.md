# middle-authorize-service-document

东莞城市学院易班学生工作站

中间授权服务说明文档

## 项目相关技术

- 开发语言：Kotlin
- JDK 版本：15
- 后端框架：
  - Spring Boot 2.4.2
  - MyBatis
- 数据库：
  - Redis
  - MySQL 8
- 项目构建工具：Gradle

## 前置知识

使用该服务时，你需要先了解：

- [OAuth 2.0](https://oauth.net/2/)
- [PKCE Flow](https://developer.okta.com/blog/2019/08/22/okta-authjs-pkce/?utm_campaign=text_website_all_multiple_dev_dev_oauth-pkce_null&utm_source=oauthio&utm_medium=cpc)
- [SHA-256](https://en.wikipedia.org/wiki/SHA-2)
- [易班开放平台](https://o.yiban.cn/)

## 描述

中间授权服务为东莞城市学院易班学生工作站集成中间服务（integrated-middle-service）中可对外开放的一部分。

中间授权服务最初目的为解决 Web 项目接入易班开发平台时，由于开放平台填写的回调接口固定，导致项目申请上线后，回调接口无法满足线上使用与线下开发同时进行的问题。

后服务加入了错误消息收集接口来收集应用服务内部引发的错误，以便根据用户反馈的错误码进行错误追踪。

## 使用服务前

使用本服务前，你需要在东莞理工学院城市学院易班学生工作站集成中间服务的管理后端登记你的应用，如同你在易班开放平台创建你的接入应用一样，如需登记你的个人应用，请联系管理员 (korilin.dev@gmail.com)

登记完成后，你需要将开发平台上的回调地址修改成中间授权服务提供的回调地址：
- 地址：https://csxy-yiban.cn/ims/callback/{appId}
- 其中`{appId}`更换为你的应用的 AppID

## 中间授权流程设计

考虑前后端分离时，前后端的开发工作不同：
- 前端需要开发用户界面，以及根据用户登录状态以及登录操作来进行页面的显示和切换
- 后端主要是提供数据接口给前端，其过程可能需要使用到 access_token，但可以使用[易班开放平台接口模拟器](https://o.yiban.cn/debug/apitest)提供的虚拟 access_token

**在我们的技术部中，建议**将 access_token 存储在前端 (vuex 或 localSession) 中作为用户登录标识，并在向后端请求数据，后端需要使用到用户令牌时，将 access_token 作为参数一同发送给后端。

也因此，将中间授权服务要在授权过程各个重定向后依旧能准确安全地将 access_token 传给前端，所以采用了 PKCE 授权流的思想。

在本服务中应用的流程设计如下：
1. 在应用生成一个随机值 v 并存储到浏览器
2. 通过哈希算法加密后得到密钥 $
3. 重定向到服务的中间授权接口
4. 中间授权服务将 $ 作为 state 重定向到易班授权接口
5. 易班接口回调到中间授权服务时，服务获取 $
6. 中间授权服务将 $ 和 code/access_token 以键值的方式存储起来，重定向到应用
7. 应用通过 POST 请求发送随机值 v 给中间授权服务
8. 中间授权服务以同样的哈希算法加密得到同样的 $
9. 服务匹配对应 $ 的 code/access_token 并返回给应用

该流程的好处是，暴露在 URL 中的只有 $，而随机值 v 只存储在用户的浏览器中，对于相同的 v 进行相同的哈希算法可以得到相同的 $，而 $ 无法被解码得到准确的 v，因此保证了 v 只有用户拥有，而中间授权服务可以根据 v 准确地匹配到 access_token 给用户

由于相同的 v 会得到相同的 $，因此不同用户需要保证 v 的唯一性，对于我们来说，无法完全相信交给应用该唯一性能得到保证，并且有多个应用使用我们的服务，这可能会导致服务内部 $ 的覆盖。

**因此我们提供了用户分配接口`/userDistribute`来分配一个 uid 给用户，代替应用自己生成随机值 v，并将该 uid 作为用户标识。**

我们不建议使用 access_token 来作为用户标识，它只能作为一个临时的用户登录状态，因为它会过期失效，并且长时间存放 access_token 可能会增加泄露的危险性。

**并且我们规定，加密算法统一采用 SHA-256 算法。**

本服务本身就是为了解决线上线下的回调接口无法共用的问题，为解决这个问题，中间授权服务提供一个统一的回调接口`/callback/{appId}`接收易班回调数据，再根据 appId 对于的应用重定向到应用登记的地址。

在重定向至中间授权服务的授权接口`/authorize`时，我们提供`source`参数来判断该授权请求的发起环境，服务在易班回调后，根据原本传入的`source`决定重定向的应用地址，其取值即重定向操作如下：
- source=0：重定向到登记的线下开发地址
- source=1：重定向到登记的线上地址

*当你在线下开发调试时，你传入的 source 应当为 0，而当你应用上线时，该值应当为 1*

## 建议的中间授权流程

所使用的接口请参考：[接口API](#接口api)

### 用户标识检查

**检查浏览器 (localStorage) 是否拥有用户标识 (uid)：**
- 没有 uid：说明用户第一次进入应用，需要向`/userDistribute`接口发送用户分配请求，获取用户标识存放在浏览器
- 拥有 uid：向`/requestToken`接口发送授权凭证换取请求，获取 access_token

**access_token 换取失败时，说明用户授权已失效，需要重新进行用户登录授权**

### 用户登录授权

**使用中间授权服务进行用户授权登录流程：**
1. 使用 SHA256 加密算法，对 uid 进行加密获得 encrypted_uid
2. 根据本次发送请求的环境源，定义好 source，线下开发为 0，线上请求为 1
3. 获取应用的 appId，将 appId、encrypted_uid、source 作为参数，重定向至接口`/authorize`进行授权

### 用户退出登录

当用户需要退出登录时，为保证用户再次授权不会直接通过授权回到应用，而是可以回到易班授权界面（用户可能需要切换账号登录），应用需要向易班发送一个授权注销请求，该请求可以通过中间授权服务的`/revokeToken`接口，或在应用后端自行进行授权回收。

在本服务中 uid 可作为客户端标识，也可以作为用户标识，当你认为该值代表一个用户时，用户退出登录后你可以移除浏览器中的 uid，以代表用户退出登录状态

## 接口 API

**用户分配接口**

- 接口地址：https://csxy-yiban.cn/ims/userDistribute
- 请求类型：GET
- 返回值：uid 字符串

**授权重定向接口**

- 接口地址：https://csxy-yiban.cn/ims/authorize
- 请求类型：GET 重定向

| 参数 | 传参方式 | 是否必填 | 描述 |
| -- | -- | :--: | -- |
| appId | URL 参数 | 是 | 应用 AppID |
| encryptedUid | URL 参数 | 是 | SHA-256 算法加密的 uid |
| source | URL 参数 | 是 | 请求来源（0 线下，1 线上）|

- 返回值：重定向到易班授权接口 / 错误接口

**客户端 Access Token 获取接口**

- 接口地址：https://csxy-yiban.cn/ims/requestToken
- 请求类型：POST form-data

| 参数 | 传参方式 | 是否必填 | 描述 |
| -- | -- | :--: | -- |
| uid | form-data | 是 | 用户标识 |

- 返回值：JSON 数据

数据格式：

```js
{
   status: 获取结果, // true or false
   accessToken: 用户令牌, // 失败时为 ""
   message: 错误消息 // 成功时为 "SUCCESS"
}
```

**注销用户授权接口**

- 接口地址：https://csxy-yiban.cn/ims/revoke
- 请求类型：POST form-data

| 参数 | 传参方式 | 是否必填 | 描述 |
| -- | -- | :--: | -- |
| uid | form-data | 是 | 用户标识 |
| accessToken | form-data | 是 | 用户授权凭证 |

- 返回值：JSON 数据

数据格式：

```js
{
   status: 处理结果, // true or false
   message: 操作信息
}
```
