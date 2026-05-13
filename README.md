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
xray uuid
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
    map $ssl_preread_server_name $backend_name {
        hostnames; # 必须添加这一行，才能解析 your domain

        .yourdomain.com            127.0.0.1:4443; 
        default                  127.0.0.1:7443;
    }

    server {
        listen 443;
        listen [::]:443;
        ssl_preread on;
        proxy_pass $backend_name;
        proxy_protocol on; # 注意：后端 4443 端口的 Nginx 也必须开启 proxy_protocol 接收
    }
}
```

说明：

- `proxy_pass` 把 443 的 TCP 流量转给本机的 Xray
- `proxy_protocol on` 用于把真实客户端地址传给后端
- 由于模板里已经设置了 `"acceptProxyProtocol": true`，前后端要保持一致

### Nginx后端设置

```nginx
server {
    # 1. 监听指令
    listen 4443 ssl proxy_protocol;
    listen [::]:4443 ssl proxy_protocol;

    server_name 业务1.你的域名.com;

    # 2. 独立开启 HTTP/2 (1.25+ 新格式)
    http2 on;

    # 3. 信任本地转发，找回真实访客 IP (保持不变)
    set_real_ip_from 127.0.0.1;
    real_ip_header proxy_protocol;

    # ... 你的证书配置和其他业务路由逻辑
    ssl_certificate /path/to/your/fullchain.cer;
    ssl_certificate_key /path/to/your/private.key;
}
```

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
- `/usr/local/etc/xray/config.json`

## 9. 客户端需要的参数

客户端连接时通常需要：

- 服务器地址：你的公网域名或 IP
- 端口：`443`
- UUID：与你的服务端一致
- Reality 公钥：服务端 `xray x25519` 输出的 `Public key`
- `shortId`：与你的服务端一致
- `serverName`：与你服务端配置一致
- `flow`：`xtls-rprx-vision`

## 10. 网站分流+warp解锁流媒体和AI

可以参照本项目下的`config.with-warp-routing.json`配置文件模板

**配置关键点说明**

1. **生成wgcf配置文件**
   
   1. **下载wgcf**
      
      首先，你需要从 wgcf 的官方 GitHub Releases 页面下载对应你系统架构的版本。
      
      对于常见的 Linux VPS（如 x86_64 架构）：
      
      ```bash
      # 下载预编译好的文件
      wget -O wgcf https://github.com/ViRb3/wgcf/releases/download/v2.2.30/wgcf_2.2.30_linux_amd64
      
      # 赋予执行权限
      chmod +x wgcf
      
      # 移动到系统目录方便全局调用（可选）
      mv wgcf /usr/local/bin/wgcf
      ```
   
   2. **注册 WARP 账户**
      
      在终端中运行以下命令，向 Cloudflare 申请分配一个免费的 WARP 账户：
      
      ```bash
      wgcf register
      ```
      
       *运行后，终端会提示你是否同意服务条款（Type `Yes`），随后会在当前目录下生成一个 `wgcf-account.toml` 文件，里面保存了你的账户密钥信息。*
      
   3. **生成 WireGuard 配置文件**
         
      账户注册成功后，直接运行生成命令：
      
      ```bash
      wgcf generate
      ```
      
       *运行成功后，当前目录下会生成你需要的 **`wgcf-profile.conf`** 文件。*
      
      ```conf
              [Interface]
      PrivateKey = yGxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx1E=  <-- 提取填入 Xray 的 secretKey
      Address = 172.16.0.2/32                                 <-- Xray address 数组的第一个值
      Address = 2606:4700:110:xxxx:xxxx:xxxx:xxxx:xxxx/128    <-- 提取填入 Xray address 数组的第二个值
      DNS = 1.1.1.1
      MTU = 1280
      
      [Peer]
      PublicKey = bmXOC+F1FxEMF9dyiK2H5/1SUtzH0JuVo51h2wPfgyo=
      AllowedIPs = 0.0.0.0/0
      AllowedIPs = ::/0
      Endpoint = engage.cloudflareclient.com:2408
      ```
      
       你只需要把 `[Interface]` 下的 **PrivateKey** 和两个 **Address**（一个 IPv4，一个 IPv6）复制出来，按照上一问我提供的 JSON 格式填入 Xray 的 `wireguard` outbounds 即可。剩下的 `[Peer]` 信息在 Cloudflare WARP 中是全球固定的，不用修改。

2. **WARP (WireGuard) 出站配置映射**
   
   你需要打开 `wgcf` 生成的 `wgcf-profile.conf` 文件，提取以下信息填入配置：
   
   - `"secretKey"`: 对应 `[Interface]` 下的 `PrivateKey`。
   
   - `"address"`: 对应 `[Interface]` 下的 `Address`。保留 `172.16.0.2/32`，并将 IPv6 地址一并填入。
   
   - `"publicKey"` 和 `"endpoint"`: Cloudflare WARP 的服务端公钥和接入点通常是固定的（已在代码中填好），直接保持原样即可。
     
     *如果服务器没有ipv6网络，`endpoint`可能需要填`162.159.192.1:2408`，而不是域名*
   
   - `"mtu": 1280`: WARP 的标准 MTU 值，不要修改。

3. **路由兜底逻辑**
   
   Xray 路由采用**自上而下匹配**，且默认命中失败的流量会走 `outbounds` 数组里的第一个出口。在这个配置中，未命中 `warp` 和 `blocked` 规则的普通流量，会自动落在第一个出站 `direct`（即服务器真实 IP 直连）。同时加入了一条屏蔽 `geoip:private` 的规则，防止服务器被利用作为内网扫描跳板。
