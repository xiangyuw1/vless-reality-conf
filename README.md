# VLESS + Reality 模板配置说明

这是一个用于参考和自建的 VLESS + Reality 配置模板。当前仓库里的 `config.template.json` 只保留了占位符,真正部署时，需要把 UUID、Reality 私钥、shortId，以及 `dest` / `serverNames` 按自己的环境替换掉。

## 1. 部署前准备

你需要准备：

- 一台可以公网访问的服务器
- 一个可用域名，或者一个能作为 `dest` 的真实 TLS 站点
- 已安装的 `xray-core`
- Xray 安装脚本（官方社区常用）：https://github.com/XTLS/Xray-install
- 已安装的 `nginx`，并确认编译/启用了 `stream` 模块
- 放行 443 端口，必要时也放行 80 端口用于证书或跳转

## 2. 生成 UUID

客户端 `id` 用 UUID。可以在 Linux 上用：

```bash
uuidgen
```

如果你在 Windows PowerShell 里生成：

```powershell
[guid]::NewGuid().ToString()
```

把生成结果填到 `config.json` 里的 `clients[0].id`。

## 3. 生成 Reality 密钥对

Reality 需要一对密钥。通常用 `xray x25519` 生成：

```bash
xray x25519
```

输出里会包含：

- `Private key`：填到服务端 `realitySettings.privateKey`
- `Public key`：填到客户端配置里

注意：`privateKey` 只应该保存在服务端，不能提交到公开仓库。

## 4. 生成 shortId

`shortId` 一般使用随机十六进制字符串，常见长度是 8 到 16 个十六进制字符。比如生成 8 字节：

```bash
openssl rand -hex 8
```

把生成结果填到：

```json
"shortIds": [
  "你的 shortId"
]
```

## 5. 修改配置文件

把 `config.json` 里的占位符替换掉：

- `id` 替换成你的 UUID
- `privateKey` 替换成 Reality 私钥
- `shortIds` 替换成你的 shortId
- `dest` 替换成一个真实可访问的 TLS 站点，例如 `www.example.com:443`
- `serverNames` 填成和 `dest` 对应的 SNI 域名

当前模板的入站监听在 `127.0.0.1:7443`，意思是 Xray 只监听本机端口，由前面的 Nginx 把外部流量转发进来。

## 6. Nginx 前置转发

因为这个模板前面套了一层 Nginx，所以 Nginx 的作用是把公网 `443` 的 TCP 流量转发到本机 `7443`。对于 Reality 来说，这里不是普通的 HTTP 反代，而是 TCP 层转发。

一个常见的 `stream` 配置示例：

```nginx
stream {
    upstream xray_reality {
        server 127.0.0.1:7443;
    }

    server {
        listen 443;
        proxy_connect_timeout 10s;
        proxy_timeout 1h;
        proxy_pass xray_reality;
        proxy_protocol on;
    }
}
```

说明：

- `proxy_pass` 把 443 的 TCP 流量转给本机的 Xray
- `proxy_protocol on` 用于把真实客户端地址传给后端
- 由于模板里已经设置了 `"acceptProxyProtocol": true`，前后端要保持一致

如果你的 Nginx 版本或编译选项不支持 `stream` / `proxy_protocol`，就不要硬套这一段，改成普通 TCP 转发或关闭 `acceptProxyProtocol`，两边必须一致。

## 7. 不使用 Nginx 转发时

如果你不打算使用 Nginx 前置转发，而是让 Xray 直接对外监听 `443`，可以按下面最小改动处理：

- 把入站 `listen` 从 `127.0.0.1` 改成 `0.0.0.0`（或你的公网网卡地址）
- 把入站 `port` 从 `7443` 改成 `443`
- 关闭 `streamSettings.sockopt.acceptProxyProtocol`（改为 `false`，或直接删除 `sockopt`）

示例：

```json
"listen": "0.0.0.0",
"port": 443,
"streamSettings": {
    "sockopt": {
        "acceptProxyProtocol": false
    }
}
```

注意：

- 如果服务器上已有其他服务占用 `443`（如 Nginx/Caddy），需要先停用或换端口
- 防火墙与安全组要放行 `443/TCP`
- 客户端连接参数保持不变，仍然使用 `flow: xtls-rprx-vision` 与对应的 `serverName/shortId/publicKey`

## 8. 启动顺序

如果你使用的是“Xray 直接监听 443”的方案，可以跳过下面顺序里的 Nginx 相关步骤。

推荐顺序如下：

1. 先准备好 Nginx 配置并确认 443 端口可监听
2. 再把 Xray 的 `config.json` 放到服务目录
3. 启动或重载 Nginx
4. 启动 Xray

常见路径示例：

- `/etc/nginx/nginx.conf`
- `/etc/xray/config.json`

## 9. 客户端需要的参数

客户端连接时通常需要：

- 服务器地址：你的公网域名或 IP
- 端口：`443`
- UUID：与你的服务端一致
- Reality 公钥：服务端 `xray x25519` 输出的 `Public key`
- `shortId`：与你的服务端一致
- `serverName`：与你服务端配置一致
- `flow`：`xtls-rprx-vision`
