# nowhere-sh

[English](README.md)

Nowhere Portal 的 Linux VPS 一键部署脚本，同时支持旧版 Anywhere 兼容协议和
Nowhere v1.5 新增的 Native Vector 原生客户端协议。

脚本会安装明确指定的 Release 版本，生成 systemd 服务，并按协议版本输出正确的
`nowhere://` 或 `vector://` 客户端链接。

## 功能

- 自动识别 `x86_64` / `aarch64` 和 `gnu` / `musl` Linux 包。
- 安装 Nowhere 到 `/usr/local/bin/nowhere`。
- 配置文件保存到 `/etc/nowhere/nowhere.env`，权限为 `0600`。
- 创建并管理 systemd 服务 `/etc/systemd/system/nowhere.service`。
- 支持 `mix` / `tcp` / `udp` 监听模式。
- 支持 `tls=1` 自签临时证书和 `tls=2` PEM 证书。
- 支持速率限制、出站源地址、日志级别、Anywhere TCP pool。
- 支持 Nowhere `v1.2.4+` 的 `socks` 出站 SOCKS5 上游代理。
- 可联网读取最近 10 个 GitHub Release，通过数字选择指定版本。
- v1.4 及更早版本输出 Anywhere 链接，并按版本自动使用 `net=` 或 `up/down`。
- v1.5+ 输出 Native Vector 链接，支持 `sni`、本地 SOCKS5 入口、分离上下行
  carrier，以及最大 256 的 TCP pool。

## 协议兼容性

Nowhere v1.5 更换了线协议，当前 Anywhere 客户端不能连接 v1.5 Portal：

| 模式 | Portal 版本 | 客户端 | 客户端链接 |
| --- | --- | --- | --- |
| Anywhere 兼容模式 | v1.4.0 及更早 | Anywhere | `nowhere://...` |
| Native Vector 模式 | v1.5.0 及以后 | 同协议版本 Nowhere 二进制 | `vector://...` |

Anywhere 入口默认安装 v1.4.0，Native Vector 入口默认安装 v1.5.0。Portal 和
Native Vector 客户端必须使用兼容的协议版本。

## 系统要求

- Linux VPS，使用 systemd。
- `curl`、`tar`。
- 推荐使用 Debian、Ubuntu、Rocky Linux、AlmaLinux、CentOS Stream 等常见发行版。

## 快速安装

推荐先下载脚本再执行：

```bash
curl -fsSL https://raw.githubusercontent.com/chikacya/nowhere-sh/main/nowhere-vps.sh -o nowhere-vps.sh
chmod +x nowhere-vps.sh
sudo bash nowhere-vps.sh
```

直接运行脚本会进入数字菜单：

```text
1) 安装/重装 Anywhere 兼容版 v1.4.0
2) 安装/重装 Native Vector 版 v1.5.0
3) 快速默认安装 Anywhere 兼容版
4) 修改当前协议模式配置
5) 指定 Release 安装/切换（最近 10 个版本）
6) 启动服务
7) 停止服务
8) 重启服务
9) 查看状态
10) 查看日志
11) 打印客户端链接/命令
12) 查看 tls=1 自签证书 SHA-256
13) 卸载服务
0) 退出
```

使用 Anywhere 选择 `1`，使用 Native Vector 选择 `2`，然后一路回车即可按默认值安装。

也可以一行执行：

```bash
curl -fsSL https://raw.githubusercontent.com/chikacya/nowhere-sh/main/nowhere-vps.sh | sudo bash -s -- install-legacy
```

安装 v1.5 Native Vector 模式时，将最后的 `install-legacy` 改为 `install-vector`。

## 指定版本安装

选择菜单 `5`，或者运行：

```bash
sudo bash nowhere-vps.sh versions
```

脚本会显示最近 10 个 Release，输入数字即可安装指定版本。v1.5 以前会自动进入
Anywhere 兼容向导，v1.5 及以后会自动进入 Native Vector 向导。选定版本会保存到
`/etc/nowhere/nowhere.env`，以后修改配置和打印链接都会沿用正确协议。

## 推荐部署方式

日常使用建议准备一个域名，并使用真实 TLS 证书：

```bash
sudo NOWHERE_PUBLIC_HOST=proxy.example.com \
  NOWHERE_PORT=443 \
  NOWHERE_NET=mix \
  NOWHERE_TLS=2 \
  NOWHERE_CRT=/etc/letsencrypt/live/proxy.example.com/fullchain.pem \
  NOWHERE_TLS_KEY=/etc/letsencrypt/live/proxy.example.com/privkey.pem \
  bash nowhere-vps.sh install --yes
```

如果只是临时测试，可以使用默认的 `tls=1`：

```bash
sudo bash nowhere-vps.sh install --yes
```

注意：`tls=1` 会在 Nowhere 启动时生成临时自签证书，重启后证书会变化。长期使用请改用 `tls=2`。

## 自签证书 SHA-256

默认 `tls=1` 会生成内存自签证书。脚本在安装、重启、更新后会自动尝试输出当前证书的 SHA-256 fingerprint：

```text
当前 tls=1 自签证书 SHA-256 fingerprint：
  AA:BB:CC:...
```

也可以随时手动查看：

```bash
sudo bash nowhere-vps.sh fingerprint
```

或者进入菜单选择 `12`。

脚本会优先读取 Nowhere `v1.2.5+` 日志中的 `CERT_SHA256|...` 字段；如果没有读到，再回退到本机 TLS 探测或旧日志匹配。

因为 `tls=1` 的证书存在内存中，Nowhere 每次重启后 fingerprint 都会变化。生产环境仍建议使用 `tls=2` 配置稳定证书。

## 安装向导

选择菜单 `1` 或 `2` 会进入对应的交互向导。每一步都会显示默认值：

```text
公网域名/IP，用于客户端连接 [1.2.3.4]:
监听地址，留空表示 IPv4/IPv6 全部监听:
监听端口 [2077]:
Shared Key [随机值]:
Spec Seed [随机值]:                         # 仅 Anywhere 模式
监听模式 mix/tcp/udp [mix]:
Vector 本地 SOCKS5 监听地址 [127.0.0.1:1080]: # 仅 Vector 模式
```

如果不想自定义，全部按回车即可。最后脚本会显示配置摘要，再次回车确认应用。

## 使用 SOCKS5 上游

如果希望 Nowhere Portal 的所有出站目标流量再经过一个 SOCKS5 代理，可以配置：

```bash
sudo NOWHERE_PUBLIC_HOST=proxy.example.com \
  NOWHERE_PORT=443 \
  NOWHERE_NET=mix \
  NOWHERE_TLS=2 \
  NOWHERE_CRT=/etc/letsencrypt/live/proxy.example.com/fullchain.pem \
  NOWHERE_TLS_KEY=/etc/letsencrypt/live/proxy.example.com/privkey.pem \
  NOWHERE_SOCKS=user:pass@127.0.0.1:1080 \
  bash nowhere-vps.sh install --yes
```

`NOWHERE_SOCKS` 支持：

```text
none
host:port
user:pass@host:port
[2001:db8::10]:1080
user:pass@[2001:db8::10]:1080
```

`NOWHERE_SOCKS` 是服务端出站设置，只写入 Portal URL，不会写入 Anywhere 客户端导入链接。Anywhere 仍然只连接你的 Nowhere Portal。

## 管理命令

```bash
sudo bash nowhere-vps.sh configure
sudo bash nowhere-vps.sh versions
sudo bash nowhere-vps.sh start
sudo bash nowhere-vps.sh stop
sudo bash nowhere-vps.sh restart
sudo bash nowhere-vps.sh status
sudo bash nowhere-vps.sh logs
sudo bash nowhere-vps.sh link
sudo bash nowhere-vps.sh fingerprint
sudo bash nowhere-vps.sh uninstall
```

常用命令说明：

- `configure`：修改当前协议模式配置并重启服务。
- `versions`：列出最近 10 个 Release，安全切换版本和协议模式。
- `update --version vX.Y.Z`：直接安装指定 Release，并使用对应协议模式。
- `logs`：实时查看 systemd 日志。
- `link`：重新打印 Anywhere 或 Native Vector 客户端链接。
- `fingerprint`：查看当前 `tls=1` 自签证书的 SHA-256 fingerprint。
- `uninstall`：删除二进制和 systemd 服务，但保留 `/etc/nowhere` 配置目录，避免误删密钥。

## 参数说明

所有参数既可以在交互模式中填写，也可以用环境变量或命令行参数传入。

| 环境变量 | 命令行参数 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `NOWHERE_PROTOCOL` | `--protocol` | `legacy` | `legacy` 为 Anywhere，`vector` 为 v1.5+ Native Vector |
| `NOWHERE_VERSION` | `--version` | 按模式 | Release 标签，例如 `v1.4.0` 或 `v1.5.0` |
| `NOWHERE_PUBLIC_HOST` | `--public-host` | 自动探测 | 客户端链接中的域名或公网 IP |
| `NOWHERE_LISTEN_HOST` | `--listen-host` | 空 | 监听地址；空表示 IPv4/IPv6 wildcard |
| `NOWHERE_PORT` | `--port` | `2077` | Portal 监听端口 |
| `NOWHERE_KEY` | `--key` | 随机生成 | Nowhere shared key |
| `NOWHERE_SPEC` | `--spec` | 随机生成 | 旧协议 seed；v1.5+ 已删除并禁止使用 |
| `NOWHERE_NET` | `--net` | `mix` | `mix`、`tcp`、`udp` |
| `NOWHERE_TLS` | `--tls` | `1` | `1` 自签临时证书，`2` PEM 证书 |
| `NOWHERE_CRT` | `--crt` | 空 | `tls=2` 的证书链路径 |
| `NOWHERE_TLS_KEY` | `--tls-key` | 空 | `tls=2` 的私钥路径 |
| `NOWHERE_ALPN` | `--alpn` | `now/1` | TLS/QUIC ALPN |
| `NOWHERE_RATE` | `--rate` | `0` | 客户端到目标方向 Mbps 限速，`0` 关闭 |
| `NOWHERE_ETAR` | `--etar` | `0` | 目标到客户端方向 Mbps 限速，`0` 关闭 |
| `NOWHERE_DIAL` | `--dial` | `auto` | 出站源 IP 或 `auto` |
| `NOWHERE_SOCKS` | `--socks` | `none` | SOCKS5 出站上游 |
| `NOWHERE_LOG` | `--log` | `info` | `none`、`debug`、`info`、`warn`、`error`、`event` |
| `NOWHERE_POOL` | `--pool` | `5` | Anywhere 范围 `0..9`，Native Vector 范围 `0..256` |
| `NOWHERE_VECTOR_SOCKS` | `--vector-socks` | `127.0.0.1:1080` | Native Vector 本地 SOCKS5 入口 |
| `NOWHERE_VECTOR_SNI` | `--sni` | `none` 或证书域名 | Native Vector 证书校验名称 |

## 防火墙

如果使用默认 `NOWHERE_NET=mix`，需要同时放行 TCP 和 UDP：

```bash
sudo ufw allow 443/tcp
sudo ufw allow 443/udp
```

firewalld 示例：

```bash
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --permanent --add-port=443/udp
sudo firewall-cmd --reload
```

如果 `NOWHERE_NET=tcp`，只需要开放 TCP。如果 `NOWHERE_NET=udp`，只需要开放 UDP。

## 客户端链接

### Anywhere 模式

安装完成后脚本会打印：

- `nowhere://...`
- `anywhere://add-proxy?link=...`

Nowhere v1.3-v1.4 的 Anywhere 链接使用 `up=` 和 `down=`：

- `up=udp&down=udp`：QUIC/UDP，UDP 可通时推荐使用。
- `up=tcp&down=tcp`：TLS/TCP fallback，带配置的 TCP pool。
- `up=tcp&down=udp` 和 `up=udp&down=tcp`：非对称 carrier 链接，只会在
  `NOWHERE_NET=mix` 且未配置 SOCKS5 上游时打印。

v1.2.x 会自动生成旧的 `net=` 参数，v1.3-v1.4 会生成 `up=` / `down=`。

在 iPhone、iPad 或 Apple TV 上，可以复制 `nowhere://` 链接到 Anywhere 中导入；如果系统能识别 Anywhere deep link，也可以直接打开对应的 `anywhere://add-proxy?link=...`。

如果忘记保存链接，在 VPS 上运行：

```bash
sudo bash nowhere-vps.sh link
```

### Native Vector 模式

v1.5+ 会输出 `vector://` URL 和客户端启动命令，例如：

```bash
nowhere 'vector://shared-key@relay.example:2077?up=udp&down=udp&sni=relay.example&socks=127.0.0.1%3A1080'
```

在客户端设备安装兼容的 v1.5+ Nowhere 二进制并执行该命令，然后让应用连接本地
SOCKS5 端口。`tls=1` 默认使用 `sni=none`，即不校验证书；正式使用建议配置
`tls=2` 的可信证书，并设置明确的域名 SNI。

## 配置文件位置

```text
/usr/local/bin/nowhere
/etc/nowhere/nowhere.env
/etc/systemd/system/nowhere.service
```

查看当前服务：

```bash
systemctl status nowhere
```

查看配置：

```bash
sudo cat /etc/nowhere/nowhere.env
```

## 故障排查

查看日志：

```bash
sudo bash nowhere-vps.sh logs
```

检查端口监听：

```bash
ss -lntup | grep nowhere
```

重启服务：

```bash
sudo bash nowhere-vps.sh restart
```

常见问题：

- 连不上：检查 VPS 安全组、防火墙、`NOWHERE_NET` 对应的 TCP/UDP 端口是否放行。
- 证书错误：生产环境建议使用 `tls=2`，并确认 `NOWHERE_PUBLIC_HOST` 与证书域名一致。
- QUIC 不通：很多云厂商安全组默认只放 TCP，记得额外放行 UDP。
- SOCKS5 不通：确认 `NOWHERE_SOCKS` 指向的代理在 VPS 上可达，认证信息正确。

## 上游项目

- Nowhere: https://github.com/NodePassProject/Nowhere
- Anywhere: https://github.com/NodePassProject/Anywhere
