# AKDNS

``` bash
curl -fsSL https://raw.githubusercontent.com/manfly17/aktools/refs/heads/main/akdns.sh \
  -o /root/akdns.sh && \
chmod +x /root/akdns.sh && \
bash /root/akdns.sh
```

### 系统清理工具

```bash
curl -fsSL https://raw.githubusercontent.com/manfly17/Xboard-Node/refs/heads/dev/sys-clean.sh \
  -o /root/sys-clean.sh && \
chmod +x /root/sys-clean.sh && \
bash /root/sys-clean.sh
```

# AriNode

### AriNode 管理脚本

这是一个交互式管理脚本，旨在简化 AriNode
的安装、部署、配置和日常维护。提供 Sing-box / Xray 内核管理、XBoard
节点配置、TLS 证书管理、DNS 配置、备份与恢复以及服务管理等功能。

#### 下载与安装

``` bash
curl -fsSL https://raw.githubusercontent.com/manfly17/AriNode/refs/heads/main/AriNode-test.sh \
  -o /root/AriNode.sh && \
chmod +x /root/AriNode.sh && \
ln -sfn /root/AriNode.sh /usr/local/bin/ari-node && \
/root/AriNode.sh install
```

#### 再次运行

安装完成后，使用以下命令打开交互式管理菜单：

``` bash
ari-node
```

### 节点协议支持与证书判断说明

AriNode 会读取 Xboard 节点的 `Core` 字段，区分 Xray 和 Sing-box 内核，并根据节点协议、传输方式及 TLS 配置，检查协议兼容性和证书申请需求。

| 协议 | Xray | Sing-box | 当前证书判断 |
| --- | --- | --- | --- |
| VMess | 支持 | 支持 | 已处理 |
| VLESS | 支持 | 支持 | 已处理 |
| Trojan | 支持 | 支持 | 已处理 |
| Shadowsocks | 支持 | 支持 | 已处理 |
| SOCKS | 不支持 | 不支持 | 不支持 |
| HTTP | 不支持 | 不支持 | 不支持 |
| WireGuard | 未实现 | 未实现 | 不支持 |
| TUIC | 不支持 | 支持 | 已处理 |
| Hysteria 2 | 不支持 | 支持 | 已处理 |
| AnyTLS | 不支持 | 支持 | 已处理 |
| Naive | 不支持 | 不支持 | 不支持 |
| Mieru | 不支持 | 支持 | 已处理 |

---

**说明：**

- **支持**：表示当前脚本对应内核支持该协议。
- **不支持**：表示当前脚本对应内核不支持该协议。
- **未实现**：表示协议尚未在当前脚本中实现。
- **已处理**：表示脚本已纳入该协议的证书判断逻辑。
- **含插件判断**：Shadowsocks 会结合插件配置判断是否涉及 TLS 证书需求。

证书判断会结合节点协议、传输方式及 TLS/REALITY 等相关配置进行识别，而不是仅根据协议名称决定是否申请证书。

## HY1 和 VLESS 用户实时更新

默认的 `sing` 后端现在为 Hysteria v1 和 VLESS
使用稳定的认证代际标识，避免删除用户后因数组位置变化而导致认证错乱。撤销
VLESS 用户授权时，会关闭该用户的父会话以及 TCP/UDP
子会话，同时保留其他正常用户和其他节点，并将最终流量归属到原面板 UID。

后续修复还覆盖了 VMess AEAD、Trojan、经典 Shadowsocks、Xray
分发器注册以及独立 Hysteria2 的认证处理。

[适用范围与尚存问题](docs/user-revocation-followup.md)。原生流量推送在确认响应丢失后，仍无法保证严格的"恰好计费一次"。详见[复现方式与测试范围](docs/hy1-vless-user-updates.md)。

## HY2/TUIC 用户撤销与流量统计

默认的 `sing` 内核现在可以在 UUID 轮换或用户重新添加后，拒绝旧的 QUIC
认证；关闭已撤销 HY2 用户的 UDP 转发，并避免 TUIC
在处理活动数据包时发生连接类型 panic。

不可变的认证代际标识可保留正常用户的会话。TCP
计数器会累计字节数，采集器以原子方式清空已采集数据；原生流量上报失败时，会在内存中保留待重试批次。

原生推送接口没有幂等键，因此即使面板已接收上报、但响应丢失，仍可能造成重复计费；面板不可用也会导致用户过期处理延迟。固定版本的
sing-quic 数据包清理回调还存在会话 ID
与对象池之间的竞态风险，本次无法通过动态竞态检测确认。

详见[复现方式、适用范围与回归测试](docs/quic-revocation-accounting.md)。

## UDP 用户速率限制

默认的 `sing` 内核现在会对普通 UDP 转发应用用户的 `speed_limit`
限速。TCP、UDP、上下行流量以及同一节点/用户的并发会话，共用现有的限速桶，同时保留
Mbps 单位、设备数限制、计费和授权撤销逻辑。

无需额外配置出站或节点。回归测试覆盖了真实 Mieru TCP/UDP 传输，以及
VLESS、SS、HY2、TUIC 和 AnyTLS 的 UDP 限速。Xray
现在也会在两个分发路径中应用共享限速桶。

独立的 `hysteria2`
内核目前仍不支持按用户进行带宽限制。详见[实现方式、测试与适用范围](docs/udp-rate-limit.md)。

## 动态 SOCKS5 出站管理器（DSOM）

DSOM 提供经过身份验证的 SOCKS5 出口 IP 探测、IPv4/IPv6
多源一致性判断、历史记录、可选
ASN/国家信息、Gateway（网关）/Direct（直连）/Auto（自动）模式、备用网关、健康阈值和冷却机制。

它通过私有回环 SOCKS 中继复用现有 Xray/sing-box
策略；上游切换时会保留原生出站和已有连接。

在本地顶层配置 `DynamicOutbounds`，然后在 XBoard 路由中引用对应的
tag，无需修改 XBoard 源码。使用 `anctl dsom status` 查看状态；使用
`anctl dsom diagnose --host HOST --port PORT [--icmp]` 诊断指定网关。

**检测到出口 IP，并不代表该 IP 对应的住宅代理一定可用。**
Direct（直连）模式只会使用明确授权的 host:port 地址，并且必须通过 SOCKS5
身份验证和真实转发测试。缺少直连权限或无法证明直连可用时，状态会显示为
`DIRECT_UNAVAILABLE`；不会扫描端口，也不会猜测端口。

第一版仅支持 TCP CONNECT。

[配置方法、参数结构、限制与诊断](docs/dsom.md) ·
[完整本地配置示例](example/dsom.config.json)。

## 自动下发 XBoard 节点配置

插件 **0.6.0** 及更新后的 Agent 会在每次同步周期（默认 30
秒）获取最新的机器模式节点配置，只对发生变化的节点应用配置，并将私有配置快照保存在
`/etc/arinode/panel-config/`。

配置未发生变化时，不会重写文件，也不会重新加载 inbound。已有 Agent
需要先升级插件，然后执行一次
`sudo anctl upgrade`。之后在面板中修改配置，无需手动重启服务。

[配置下发机制与兼容性说明](docs/xboard-config-sync.md)。

### 面板 TLS 证书

XBoard 的 `cert_config` 内容/文件配置现在可以传递到 TLS inbound，包括
AnyTLS、HY2 和 TUIC。

证书内容会经过校验，并以私有权限保存；如果证书没有变化，则会重复使用现有文件，不会重新写入。自动生成的自签名证书也会在重新加载后保留。

针对 XBoard `4f48e61`，还需要一个小型配套补丁，用于添加 AnyTLS 的
`security=tls`、稳定配置映射，以及 SNI/SAN 保存校验。

[诊断、部署、验证与回滚说明](docs/panel-certificates.md)。

AriNode 是一个基于 V2bX 的节点服务，集成了 XBoard 节点配置和 Komari
状态监控。它保留 V2bX 的 sing-box、Xray 和 Hysteria2 内核，并使用 XBoard
原生 UniProxy 节点 API 获取配置、用户、流量和在线状态报告。

[中文说明](README.zh-CN.md)。

本仓库基于 [V2bX](https://github.com/wyx2685/V2bX) 的
`71277de69efbbc86c23ad8ae02b68efd174e5756` 提交版本开发，遵循 MPL-2.0
许可证。XBoard 适配部分参考并对照了
[Xboard-Node](https://github.com/cedar2025/Xboard-Node) 和
[Xboard](https://github.com/cedar2025/Xboard)。Komari 插件遵循 [Komari
插件 SDK](https://github.com/komari-monitor/plugin-sdk)。

## 一键安装（Linux）

在使用 systemd 的 Linux 主机上执行：

``` sh
curl -fsSL https://raw.githubusercontent.com/Foxtea267/AriNode/main/scripts/install.sh | sudo bash
```

安装程序会下载并验证最新稳定版本，然后询问使用中文还是英文，以及选择部署方式：全新安装、从现有
xbnode 迁移，或保留现有 AriNode 配置。

如果存在
`/etc/xboard-node/config.yml`，默认会选择迁移模式。迁移时会读取该配置及其旁边的
`credentials.env`，无需再次输入 XBoard 地址或令牌；同时会备份已有
AriNode 配置。如果 AriNode 启动失败，程序会尝试切换回 xbnode systemd
服务并重启。

全新安装时，程序会要求填写 XBoard
地址、令牌（输入时隐藏）、节点列表（例如
`vless:1 trojan:2`），以及可选的机器 ID。配置文件权限为 `0600`。

自动升级默认关闭，只有用户明确启用后才会开启。系统要求 Linux `amd64` 或
`arm64` 架构，并安装 `curl`、`tar`、`sha256sum` 和 `systemctl`。

执行 `sudo anctl bash` 可重新打开部署菜单。使用 `anctl status` 和
`anctl log` 检查服务状态及日志。

XBoard 插件 0.7.0 支持在尚未分配节点时先注册机器。之后 Agent
会自动拉取机器绑定关系和节点配置。启用多面板模式后，可追加面板；使用
`sudo anctl panel list` 查看面板地址，使用 `sudo anctl panel remove ID`
删除某个绑定及其已下载文件。

[XBoard 部署指南](docs/xboard-plugin.md)

## 快速开始

### XBoard 自定义出站、路由与多路复用

AriNode 会将面板中的 `custom_outbounds`、原生 `custom_routes`、结构化
`custom_route_rules`、普通 `routes` 转发规则（例如
`action: "proxy"`、`action_value: "sg01"`）以及 `multiplex`
应用到正在运行的 Xray/sing-box 内核。

出站和路由引用按节点隔离，包括代理链。仅修改路由时会自动更新，不需要重启
inbound 或共享 Core；修改多路复用配置时，只会重新加载受影响的节点。

本地配置项 `OutboundConfigPath`、`RouteConfigPath`、`OriginalPath` 和
`MultiplexConfig` 仍然受支持。

详见[HK01 → SG01
完整路由示例、优先级、协议矩阵与验证指南](docs/xboard-routing.md)。

sing-box 服务端 mux 会应用
enabled/padding/brutal；协议类型以及连接/数据流数量属于客户端/出站设置。Xray
服务端 mux 不需要对应的 inbound 字段。

当前固定版本的 sing-box 构建不支持 Naive 出站，并会明确拒绝该配置；Mieru
自定义出站使用现有 SDK。

XBoard 的 `block` 规则支持普通域名和
`*.example.com`（匹配该域名及其子域名）、IP/CIDR，以及
`domain:`、`full:`、`keyword:`、`regexp:` 和 `protocol:` 前缀。

普通域名按域名后缀匹配；正则表达式必须使用 `regexp:`
前缀。规则更新会先进行校验，再替换当前生效规则。

如果旧版本出现
`invalid domain rule`，节点持续重试且没有代理监听端口，请执行
`sudo anctl upgrade`。

1.  在 XBoard 中创建并启用节点，记录每个节点的
    ID，以及服务器设置中的全局服务器令牌。
2.  将 `example/arinode.config.json` 复制为 `config.json`，设置
    `ApiHost`、`ApiKey`、`NodeID` 和 `NodeType`。每增加一个节点，就在
    `Nodes` 中添加一个对象。
3.  执行 `docker compose up -d --build`。由于代理 inbound
    会监听主机端口，因此使用 host 网络模式。
4.  使用 `docker compose logs -f arinode` 查看日志，并在宿主机上执行
    `curl http://127.0.0.1:18086/v1/status` 检查状态。

Docker 镜像使用 Go 1.25 构建。原生编译方式：

``` sh
GOEXPERIMENT=jsonv2 go build -tags 'sing xray hysteria2 with_quic with_grpc with_utls with_wireguard with_acme with_gvisor' -o arinode .
GOEXPERIMENT=jsonv2 go build -tags 'sing xray hysteria2 with_quic with_grpc with_utls with_wireguard with_acme with_gvisor' -o anctl ./cmd/anctl
ARINODE_PANEL_TOKEN='YOUR_XBOARD_SERVER_TOKEN' ./anctl init --panel https://panel.example.com --node vless:1 --node trojan:2 --output config.json
./arinode server --config config.json
```

如需使用 systemd 运行原生二进制，请执行
`sudo install -m 755 arinode anctl /usr/local/bin/`，将受保护的配置文件复制到
`/etc/arinode/config.json`，将
[`deploy/arinode.service`](deploy/arinode.service) 放入
`/etc/systemd/system/`，然后执行：

``` sh
systemctl daemon-reload && systemctl enable --now arinode
```

之后可使用 `anctl start`、`anctl stop`、`anctl restart`、`anctl status`
和 `anctl log` 管理该服务。

## 稳定版本与升级

官方稳定版本采用 `YYYYMMDD-commit` 格式命名，例如
`20261001-abcdef12`。日期代表发布提交的 UTC 日期，后缀为对应 Git
提交的短哈希。开发构建版本显示为 `dev`。

每次向 `main` 分支推送代码时，都会运行 **Test and release AriNode**
工作流；测试通过后自动发布稳定版本。

维护者也可以在 `main` 分支手动点击 **Run workflow**，或为已经位于 `main`
分支的提交推送符合规则的标签。重复发布相同版本时，工作流只会验证现有稳定版本及其完整资源列表，不会重复发布。

工作流会发布 Linux `amd64` 和 `arm64`
压缩包，内含两个可执行文件、两个插件 ZIP 包以及 `SHA256SUMS`。

只有符合该版本格式、已正式发布且非预发布状态的 GitHub
Release，才符合升级条件。

原生 Linux systemd 安装可使用以下命令：

``` sh
anctl version
anctl upgrade check
sudo anctl upgrade
anctl upgrade auto status
sudo anctl upgrade auto enable
sudo anctl upgrade auto disable
```

自动升级**默认关闭**。执行 `auto enable` 会安装并启动每日运行的 systemd
timer；执行 `auto disable` 会移除该定时器。

定时器会检查最新稳定版本，仅当远端版本不同于当前版本且日期不早于当前版本时才执行升级。

`anctl upgrade` 会下载匹配当前 CPU 架构的版本，校验 GitHub 提供的 SHA256
资源摘要，验证两个二进制文件及其内嵌版本号，然后替换文件，并在
`arinode.service` 原本处于运行状态时重启服务。

如果替换或重启失败，程序会恢复之前的二进制文件，并尝试重新启动旧服务。配置文件不会被修改。

Docker 部署应通过原有部署流程更新镜像；systemd 定时升级功能仅适用于原生
Linux 安装。

`init` 命令会创建权限为 `0600` 的配置文件；除非提供
`--force`，否则不会覆盖已有文件。它支持重复使用 `--node type:id` 参数。

令牌也可以通过 `--token` 传入，但使用环境变量可以避免令牌进入 Shell
历史记录。

对于 XBoard 机器认证，应使用机器令牌并添加
`--machine-id ID`；所列节点必须已经在 XBoard 中绑定到该机器。

## 迁移到 XBoard-Node 或从 XBoard-Node 迁出

### 一条命令完成 V2bX 安装、迁移与卸载

``` sh
curl -fsSL https://raw.githubusercontent.com/Foxtea267/AriNode/main/scripts/install.sh | sudo bash -s -- --migrate-v2bx
```

对于已经安装 AriNode 的环境，可执行：

``` sh
sudo anctl migrate from-v2bx
```

如需迁回 V2bX，执行：

``` sh
sudo anctl migrate to-v2bx
```

完整的 V2bX 迁移流程会备份原安装、复制证书和原生 Core
资源、检查目标环境，然后卸载源运行环境。原始可执行文件会保留在受保护的恢复快照中。

使用 `--dry-run` 可预览迁移过程；对于非标准安装，可使用
`--config-only`。

迁移共享节点成员时，可添加
`--cluster-domain pool.example.com --cluster-member hk-04`。请先将
XBoard 插件升级到 0.3.0。

[迁移、回滚、兼容性与四服务器指南](docs/v2bx-migration.md)。

在同时安装了两个 systemd 服务单元的 Linux
主机上，任一方向都可以通过一条命令完成切换：

``` sh
sudo anctl migrate from-xbnode --switch
sudo anctl migrate to-xbnode --switch
```

默认配置路径为 `/etc/xboard-node/config.yml` 和
`/etc/arinode/config.json`。

导入时还会读取 xbnode 配置文件旁边的
`credentials.env`，并将其中的键值赋值按字面内容解析，以便处理令牌环境变量。

如果目标配置已经存在，可添加 `--force`；原有内容会备份为带时间戳的
`.bak-*` 文件。

使用 `--dry-run` 可检查节点数量和警告，不会写入文件，也不会输出凭据。

省略 `--switch` 时只转换配置，不切换服务；也可以使用 `--input` 和
`--output` 指定自定义路径。

导入程序会自动向 XBoard 查询 YAML
中缺失的节点类型，并发现当前机器绑定关系。离线单节点导入可使用
`--offline --node-type vless`；机器导入则需要能够访问面板。

导出的机器绑定关系使用 XBoard-Node
的动态发现机制，可能包含绑定到该机器的其他节点。

部分内核自定义配置、WebSocket、轮询和运行时设置仍需人工检查。证书路径会保留，但证书文件不会复制。

如果新 systemd 服务启动失败，命令会尝试重新启动原来的服务。

## XBoard 插件

插件 **0.9.3** 新增服务器管理页面，地址为：

`https://YOUR_PANEL/arinode/admin`

页面采用紧凑的单色表格，提供节点目录、编辑/部署对话框以及深色/浅色主题。

使用同源 XBoard
管理员账号登录后台，点击**添加服务器**，填写服务器名称，可选已有节点，然后保存。

将生成的部署命令复制到目标 Linux 服务器执行。注册码有效期为 30
分钟，AriNode 可通过该注册码获取机器凭据、绑定关系和节点配置。

默认内核为 sing-box。兼容的 XBoard 管理后台会显示原生 **Servers /
Available nodes（服务器 / 可用节点）** 插件菜单。

[页面与部署指南](docs/xboard-plugin.md)。

原生的**服务器与部署命令**表格包含一个宽度为 290 px
的命令列，每行提供**复制命令**按钮，以及一个宽度为 140 px 的部署提示列。

点击按钮会复制完整命令。注册码过期时，页面会提示刷新列表；如果浏览器禁止自动复制，则会显示已选中的命令，用户可按
Ctrl+C / ⌘C 复制。

由 Laravel 按配置的 `secure_path` 提供服务的 XBoard 管理后台
HTML，会自动加载此增强功能。

独立托管的静态管理页面，可以通过自身域名加载 `/arinode/native-table.css`
和延迟加载的 `/arinode/native-table.js`，也可以直接使用
`/arinode/admin`。

自定义 CSP 必须允许加载同源脚本和样式。

[静态前端集成说明](docs/xboard-plugin.md#原生后台表格中的部署命令)。

如果前端与 API 分开部署，请打开**通讯设置**，将**API
通讯地址**设置为后端 HTTPS 基础地址，例如 `https://api.example.com` 或
`https://api.example.com:8443/xboard`。

XBoard 原生插件设置中也提供相同的 `api_url` 字段。

明确设置的 API 地址优先级最高，其次是站点的
`app_url`，最后才是当前请求来源地址。

部署命令、原生插件服务器列表和 `/provision` 导出都会使用该地址。

填写服务基础地址即可，不要添加 `/api` 或 `/api/v1`。

该设置保存在 XBoard
插件配置中，保存后会立即应用于新生成的命令和导出内容。

[API 地址设置](docs/xboard-plugin.md#api-通讯地址)。

如需修改现有 VPS 绑定，请执行：

``` sh
sudo anctl panel list
```

使用以下命令删除旧绑定：

``` sh
sudo anctl panel remove OLD_ID
```

然后执行新生成的部署命令。

删除旧绑定后，相关节点会暂时停止，直到新注册完成，因此请预留短暂中断时间。

已有 VPS 在修改地址后，需要重新执行注册流程。

手动安装一次 0.5.0 后，可在该页面使用**插件更新**检查并安装官方插件包。

下载内容会固定到单个 Git Commit，并在备份及 XBoard
原生升级前完成校验；如果失败则会回滚。服务器绑定和密钥会保留。

需要 PHP cURL/ZIP
扩展，并确保插件目录具有写入权限。此操作只更新面板插件。

在服务器列表中点击**更新后端**，再点击**更新到最新正式版**，即可更新对应
VPS。

可以跨页面选择服务器并执行**批量更新后端**，也可以点击**全部更新后端**。

全部服务器列表会覆盖所有分页，不受当前搜索筛选条件影响；预览中也会显示已禁用的机器，以便明确看到哪些机器会被跳过。

检查预览后，点击**下发更新任务**确认。

浏览器会检查状态，并以最多 3
个并发请求将任务加入队列，然后逐台报告结果。

已禁用、离线、存在冲突或正在更新的机器会被跳过。

界面复用每台机器现有的状态和更新队列，不会额外调用批量更新后端接口。

通过 HTTPS 注册的较新 Linux systemd
安装版本，会报告任务进度；如果新服务未通过健康检查，则会尝试回滚。

较旧的 Agent 需要先执行一次 `sudo anctl upgrade`。

每台符合条件的 VPS 在安装更新时，AriNode 都会短暂重启。

[后端更新要求与日志](docs/xboard-plugin.md#手动更新-vps-后端)。

将 [`dist/arinode-xboard-plugin.zip`](dist/arinode-xboard-plugin.zip)
上传到 XBoard 的插件管理页面，然后安装并启用 `ari_node`。

也可以将 [`integrations/xboard/AriNode`](integrations/xboard/AriNode)
复制到 `<Xboard>/plugins/AriNode`。

插件提供一个仅管理员可访问的 `POST /api/v1/arinode/provision` 接口。

通过已认证的 XBoard 管理员会话发送：

``` json
{"node_ids":[1,2],"core":"sing"}
```

接口会返回所选**已存在且已启用节点**的完整 AriNode
`config.json`，包括节点 ID、节点类型和 XBoard 服务器令牌。

使用机器认证时，请先在 XBoard
中创建机器并将节点绑定到该机器，然后在请求中添加
`"machine_id":3`；插件会验证绑定关系，并返回该机器的令牌。

请将响应保存为机密配置文件，并设置权限：

``` sh
chmod 600 config.json
```

该接口不会创建或修改 XBoard 节点。

支持的绑定类型包括：VMess、VLESS、Trojan、Shadowsocks、Hysteria
1/2、TUIC v5、AnyTLS 和 Mieru。

XBoard 原生 `hysteria` 绑定在 `version: 2` 时，内部使用 Hysteria2。

Xray 绑定支持 VMess、VLESS、Trojan 和 Shadowsocks；其他协议请使用
`sing`。

传统 UniProxy 绑定使用 XBoard 全局
`server_token`；机器绑定使用机器专属令牌。

详见[协议验证矩阵](docs/protocol-compatibility.md)。

## Komari 插件

通过 Komari 的插件管理器上传
[`dist/arinode-komari-plugin.zip`](dist/arinode-komari-plugin.zip)。

该插件会增加一个管理员页面，以及一个仅管理员可访问的
`GET /api/arinode/status` 路由。

如果 Komari 运行在不同的网络命名空间中，请配置 `status_url`。

默认情况下，状态监听器绑定到 `127.0.0.1:18086`；如果需要远程监听，请设置
`ARINODE_STATUS_TOKEN`，并在 Komari 中配置相同的 `status_token`。

状态 API 会返回服务启动时间、各节点状态和 Komari
上报状态，不会暴露面板凭据。

`/healthz` 提供简单的本地就绪状态检查。

## 多台服务器共享一个节点

在多台 AriNode 服务器上使用相同的 XBoard 节点
ID，并将该节点的主机名解析到多个 A/AAAA 记录。

在每台服务器对应的节点配置中添加：

``` json
"Cluster": {"Domain":"pool.example.com","MemberID":"hk-01"}
```

每台主机都应使用不同的 `MemberID`。

请先更新并启用 **XBoard 插件
0.2.0**，以便合并流量、设备和主机指标，并生成各副本独立的配置。

详见[部署指南](docs/node-cluster.md)和[配置示例](example/node-cluster.config.json)。

DNS 和客户端负责选择后端；该机制不会按 CPU
负载加权分配，也不会自动移除故障 IP。

设备数量采用最终一致方式同步，速率限制则按每台服务器分别生效。

## 多个 XBoard 与 Komari 绑定

如果省略 `Cores`、内核 `Type` 或节点 `Core`，默认使用 sing-box。

`sing`、`singbox` 和 `sing-box` 都是 `sing` 的有效别名。

明确指定 `CoreName`
时，会选择对应名称的内核；其他内核需要显式配置并选择节点。

状态 API 会包含每个运行节点实际使用的内核和监听端口。

仅凭面板上报信息，无法验证客户端是否真正连接成功。

**xhttp / splithttp 特殊说明：** 当前内置的 sing-box
不支持这些传输方式。

如果某个 sing-box 绑定未设置 `CoreName`，系统会自动为其使用内置
Xray；如果已配置 Xray 内核，则复用现有内核，否则按需启动一个 Xray
实例，无需单独下载二进制文件。

其他传输方式仍使用指定内核，即使面板修改了传输方式也不会改变。

如果明确指定了 sing-box 内核，系统会提示该传输方式需要 Xray。

解析前会对 XBoard/PHP 中 xhttp `extra`
字段里的空对象数组进行规范化处理。

如果旧版本日志出现 `unknown transport type: xhttp`，请升级。

**客户端设置：** 服务端规范化处理不会修改客户端订阅。

如果客户端出现 `cannot unmarshal JSON array` 错误，请将空对象字段中的
`[]` 改为 `{}`，或直接省略该字段。

除非确实配置了独立下载端点，否则请删除 `extra.downloadSettings`。

地址为空、端口不匹配，或 TLS/Reality
设置不一致，都无法构成可用的同节点默认配置。

请在面板中使用[单节点 xhttp extra
示例](example/xhttp-extra.json)，保留节点原有的主机名、路径和 Reality
设置，然后刷新客户端订阅。

**Mieru：** 使用 `NodeType: "mieru"` 和 `sing` 内核（也可以省略
`Core`）。

AriNode 内置 [Mieru
v3.29.0](https://github.com/enfein/mieru/tree/v3.29.0)，支持面板中的
TCP/UDP 传输和 `traffic_pattern`，并将每位订阅用户的 UUID
同时用作用户名和密码。

该 inbound 共享 sing-box 的路由、限速和流量上报机制。

请为所选传输方式开放对应节点端口。

**过期与授权撤销：** 授权状态以 XBoard `/user`
接口返回的有效用户为准；原生 API 不提供订阅过期时间戳。

轮询间隔使用面板的 `pull_interval`，最长不超过 30
秒；此外还会受到面板缓存、响应时间和网络延迟影响。

成功撤销授权（包括用户列表为空）后，会关闭受影响 Mieru/AnyTLS 用户已有的
TCP/UDP 会话，并拒绝新连接，以及复用传输中的新数据流，同时保留正常用户。

授权撤销会在节点配置请求/重新加载之前应用；即使重新加载失败，也不会恢复已删除的用户。

如果内核更新失败，会使用户 ETag 失效，以便重新拉取完整快照。

如果面板不可用，则继续沿用最后一次有效授权状态。

**机器自动发现：** 机器绑定会在启动时以及每隔 30 秒自动请求
`/api/v2/server/machine/nodes`。

在 XBoard 中新增的节点会自动启动；收到有效响应后，未绑定的节点会停止。

配置未发生变化的节点会保留现有连接；如果发现请求失败或响应格式错误，则保留原有绑定。

显式配置的本地节点条目可以设置各节点选项；新发现的条目会继承该机器第一个本地绑定的选项，并获得唯一的运行时
tag。

如果希望只管理显式配置的 `Nodes` 条目，请在该机器的每个本地条目中设置
`MachineAutoDiscover: false`。

运行时发现不会改写 `config.json`。

**TLS：** 如果省略 `CertConfig`，则使用 `CertMode: "auto"`。

对于面板明确设置 `tls_settings.allow_insecure: true` 的 TLS
节点，自动模式会创建内存中的自签名证书。

如果客户端需要验证证书，则必须显式配置 `file`、`http`、`dns` 或 `self`
证书方式；在需要验证的情况下，自动模式会报告配置错误。

已有的显式证书设置会保留。

TUIC 使用面板提供的 ALPN 列表；如果未设置，则默认使用 `h3`。

同时支持原生 PHP 中的空 network 对象/headers，以及数字或字符串格式的
Reality 字段。

使用
[`example/multi-bindings.config.json`](example/multi-bindings.config.json)，可以在
`Panels` 中配置多个 XBoard 面板，并在 `Nodes` 中引用各自的面板。

每个绑定都可以使用独立的服务器令牌或机器令牌。

不同面板中具有相同机器 ID 的绑定，会分别上报各自的机器状态。

可添加多个 `Komari` 配置项，每个配置项设置唯一的 `Name`、面板
`Endpoint`，以及客户端 `Token` 或 `TokenEnv`。

AriNode 会通过 Agent v2 HTTP RPC，分别向多个 Komari 面板上报主机指标。

该功能仅提供监控数据上报；Komari 远程命令和 WebSSH 不属于此集成范围。

现有 Komari 插件可安装到每个面板，用于查看 AriNode 本地状态 API。

XBoard 节点故障时，会每隔 30 秒重试，不会影响正常节点。

配置重新加载时，只会重启发生变化的绑定。

`/v1/status` 会公开每个绑定的 `running` 或 `retrying` 状态，以及 Komari
上报状态。

内核配置发生变化时，需要执行 `anctl restart`。

隔离机制覆盖每个节点的配置、面板请求、启动过程、重新加载错误和后台任务
panic。

不过，所有绑定仍然共用一个 AriNode
进程；如果该进程发生致命崩溃，仍需为每个节点运行独立进程，才能实现进程级隔离。

## TCP 优化

在 Linux 主机上执行：

``` sh
sudo bash scripts/tune.sh bbr
sudo bash scripts/tune.sh tcpfit
sudo bash scripts/tune.sh status
```

`bbr` 会检查内核支持情况，写入 `/etc/sysctl.d/90-arinode-bbr.conf`，应用
BBR + fq，并验证 BBR 是否生效。

`tcpfit` 会下载固定版本的上游 `v0.3.8`，校验其 SHA256
清单，然后启动上游安装程序/菜单。

这些操作会修改宿主机系统，应在宿主机上执行，不要在容器内部运行。

[tcpfit
项目](https://github.com/Kylin010/tcpfit)负责其自身的网络调优逻辑。

## 当前兼容性边界

AriNode 使用与 Xboard-Node 相同的 XBoard **UniProxy 节点模式 API**：

-   `/api/v1/server/UniProxy/config`
-   `/api/v1/server/UniProxy/user`
-   `/api/v1/server/UniProxy/alivelist`
-   `/api/v1/server/UniProxy/push`
-   `/api/v1/server/UniProxy/alive`
-   `/api/v1/server/UniProxy/status`

对于分配给机器的节点，AriNode 会使用相应的 `/api/v2/server/*`
接口，自动发现机器绑定关系，并通过 `/api/v2/server/machine/status`
上报主机负载。

目前，这个 V2bX 分支尚未实现 Xboard-Node 的 WebSocket
推送、握手机制和合并上报 API。

Komari 客户端令牌必须在对应面板中创建；AriNode
不会自动创建受监控客户端。

## 许可证

MPL-2.0，详见 [LICENSE](LICENSE)。

本仓库保留上游项目的版权归属声明。
