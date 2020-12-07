# unified-authorization

易班统一授权服务

### [本服务适用于](#本服务适用于)

前后端分离的项目，前后端分开部署。

前端部署后不中断部署，采用覆盖的方式更新应用。

### [易班登陆流程](#易班登陆流程)

1. 调用授权接口获取用户令牌（code，授权码）
   - 接口 URI：https://openapi.yiban.cn/oauth/authorize
   - 请求方式：GET 重定向
   - 参数 client_id = AppID
   - 参数 redirect_uri = 授权回调地址（站内应用填写站内地址）
   - 防跨站伪造参数 state
2. 易班授权服务器向回调地址进行响应
   - 响应参数 code = 授权码，有效期 300 秒
   - 防跨站伪造参数 state
3. 调用授权凭证接口获取授权凭证（access_token）
   - 接口 URI：https://openapi.yiban.cn/oauth/access_token
   - 请求方式：POST 请求（form-data 方式）
   - 参数 client_id = appID
   - 参数 client_secret = AppSecret
   - 参数 redirect_uri = 授权回调地址（站内应用填写站内地址）
   - 参数 code = 授权码

详见：[易班开发平台wiki文档授权机制说明](https://o.yiban.cn/wiki/index.php?page=%E6%8E%88%E6%9D%83%E6%9C%BA%E5%88%B6%E8%AF%B4%E6%98%8E)

### [易班接口限制与凭证说明](#易班接口限制与凭证说明)

详见：[易班开发平台wiki文档权限说明](https://o.yiban.cn/wiki/index.php?page=%E6%9D%83%E9%99%90%E8%AF%B4%E6%98%8E)

每个应用获得同一用户的授权凭证独立，每个应用的接口调用数据相互独立，因此不同项目应分开在易班开发平台创建接入和进行审核

### [本服务处理的问题](#本服务处理的问题)

在易班开放平台创建接入的应用，在开发时需要将回调地址修改为本地地址，而在生产环境中需要修改为服务地址。

应用部署到生产环境后，需要修改服务代码时，需要将应用设置为维护状态，修改回调地址，才能在本地修改代码。

学生可能直接使用链接进入应用，而不是在易班应用广场进入，导致无法跳转至维护地址，出现页面无法访问情况。

**本服务处理的问题**

- 统一一个线上服务处理不同应用的授权登陆（本地开发需联网），本地开发与生产环境使用同一回调地址，需要修改代码应用时不需要进入维护状态
- 可在本服务设置维护状态与维护地址，出现访问的应用为维护状态时，直接跳转到维护页面

### [使用方法与API](#使用方法与API)

[请参考 unified-authorization-service 仓库](https://github.com/csxyyiban/unified-authorization-service)
