# HY2 / WG-NAT / 端口限速 / 可选 SOCKS5 落地

命令使用 **root + Bash**，示例端口换成实际端口。日常查询直接看表格，长命令和低频操作点击展开；所有具体用法均在本 README 内。

[查询](#queries) · [新建节点](#create) · [修改与重置](#modify) · [排查](#trouble-guide) · [封禁与删除](#remove) · [安装与更新](#install)

HY2 主节点使用 UDP/QUIC 与正式 TLS。普通临时节点、WG-NAT 临时节点、配额、来源 IP 和限速由 HY2 VPS 管理；NAT 出口机和可选 SOCKS5 落地机**独立部署**。

<a id="queries"></a>

## 一、查询与检查

| 查询什么 | 命令 / 直接跳转 |
|---|---|
| HY2 主节点、到期、临时节点配额及 IP 占用 | `hy2_audit.sh` |
| 指定端口节点链接 | [按端口查链接](#node-link) |
| 指定端口临时节点状态与日志 | [按端口查状态与日志](#port-log) |
| Hysteria 核心版本（只读） | `hysteria version \| grep '^Version:'` |
| 下次配额重置日期与倒计时 | [查重置时间](#quota-reset-time) |
| HY2 今天 / 某端口每日流量 | `hy2_traffic` / `hy2_traffic 40020` |
| 全部端口限速 | `portbw report` |
| 自动降速与恢复日志 | [查限速日志](#speed-logs) |
| 当前来源 IP 与 UDP 监听 | [查来源 IP](#active-ips) |
| 共享线索、异常日志与可选实时 TCP 目标 | `PORT=40020 DAYS=7 hy2-abuse`（[首次安装](#abuse-install)） |
| 全部受管理黑名单与端口暂停 | [查黑名单](#blacklist-report) |
| 全部来源 IP 白名单 | [查白名单](#source-allowlist) |
| SOCKS5 账号、有效期、总配额和来源占位 | 在**独立落地机**执行 `socks5 list` |
| SOCKS5 每日流量、链接和访问审计 | [每日用量](#socks5-daily) / `socks5 link 41004` / [审计](#socks5-access) |
| NAT 出口机运行状态 | 在 NAT 机执行 `bash /root/nat.sh status` |
| 导出有效临时节点订阅 | [导出订阅](#subscription) |
| 按端口删除某一个临时节点 | [按端口删除](#node-manage) |
| 配额、限速采样进度等精确状态 | [按需补充查询](#advanced-queries) |

每日流量与配额余额含义不同；来源 IP 槽位数不等于实际用户数。示例端口需替换，节点密码、私钥和原始日志不要公开。

<a id="advanced-queries"></a>

<a id="socks5-query"></a>

<details>
<summary>补充查询：只在总览未提供所需信息时使用</summary>

SOCKS5 的 LIMIT / USED / LEFT 是账号的总配额、已用量和剩余量；TTL 表示剩余有效期。IP 占位数不等于 TCP 连接数；读取失败不能视为零。

| 什么时候用 | 命令 | 比总览多什么 |
|---|---|---|
| 配额归属、下一周期、精确总用量 | `pq_audit.sh 40020` | 按端口查看受管配额和重置时间 |
| 查看该端口静态限速 | `portbw show 40020` | 本机单端口限速配置 |
| 自动限速进度与保护时间 | `portbw auto status 40020` | 当前采样状态 |
| 限速配置、内核规则是否一致 | `portbw audit` | 异常时返回错误状态 |
| SOCKS5 精确配额字节 | `socks5 quota 41004` | 账号总额与剩余额度 |
| SOCKS5 完整凭据（敏感） | `socks5 show 41004 --credentials` | 包含密码的账号详情 |
| SOCKS5 出口、路由与服务 | `socks5 status` | 运行状态 |
| SOCKS5 服务和防护检查 | `socks5 audit 41004` | 不等于历史访问审计 |
| HY2 UDP、TLS、WG 与状态检查 | `hy2_doctor.sh --strict` | 系统诊断，不能代替真实客户端 |

`pq_audit.sh` 与 `hy2_audit.sh` 对同一节点使用相同配额来源，不是两套计费。SOCKS5 的所有命令都只在独立落地机执行。

</details>

<a id="node-link"></a>

<details>
<summary>按端口查询 HY2 节点链接（只改 PORT）</summary>

在 **HY2 VPS** 的 root Bash 中执行。仅修改第一行的 `PORT=40020`，程序会从现有主节点元数据读取实际监听端口，不假定主节点永远是 443。返回的 `hy2://` 链接包含密码，请勿公开。

```bash
PORT=40020
(
  set -Eeuo pipefail
  [[ "$PORT" =~ ^[1-9][0-9]{0,4}$ ]] && (( 10#$PORT <= 65535 )) || {
    echo "无效端口" >&2
    exit 1
  }
  source /usr/local/lib/hy2/common.sh
  if [[ "$PORT" == "$(hy2_main_port)" ]]; then
    LINK=/root/hy2_main_url.txt
  else
    META="$(hy2_temp_meta_by_port "$PORT")"
    [[ -n "$META" && -f "$META" ]] || hy2_die "未找到此端口的受管临时节点"
    TAG="$(hy2_meta_get "$META" TAG)"
    LINK="$(hy2_temp_url_file "$TAG")"
  fi
  [[ -s "$LINK" ]] || hy2_die "链接文件不存在或为空：$LINK"
  cat "$LINK"
)
```

没有匹配节点、同端口存在多个不一致元数据、链接文件缺失时应报错并停止；不要通过猜测 TAG 或拼凑密码来生成链接。

</details>

<a id="active-ips"></a>

<details>

<summary>查看来源 IP 占位与 UDP 监听</summary>

`IP_LIMIT` 为活跃来源 IP 槽位，不代表实际人数。HY2 使用 UDP/QUIC，此处检查 UDP 监听，不提供 TCP `established` 连接清单。

```bash
PORT=40020
source /usr/local/lib/hy2/iplimit-lib.sh
hy2_il_active_ips "$PORT"
ss -lunp | grep ":$PORT " || true

```

</details>

<a id="daily-traffic"></a>

<details>
<summary>HY2 每日流量：今天、单端口最近 30 天、历史 JSON 导出</summary>

**每日流量统计为独立可选组件**：使用 HY2 UDP 双向计数器，并识别主节点及临时节点。按北京时间自然日保存最近 30 天；`UP` 为用户上传，`DOWN` 为用户下载，`TOTAL` 为当日合计。此处记录的是**每日流量历史**，不同于 `pq_audit.sh` 的双向总配额。必须先按 [6.3 安装每日流量](#daily-install)，安装前的历史流量不能补记。

```bash
# 所有当前节点今天的上传、下载
hy2_traffic

# 指定临时节点最近 30 天
hy2_traffic 40020

# 主节点最近 30 天
hy2_traffic main

# 按临时节点 ID 或完整 TAG
hy2_traffic nat4
hy2_traffic hy2-temp-nat4

# 导出指定端口仍保留的每日历史 JSON（字节）
hy2_traffic 40020 --json
```

- 数据文件 `/var/lib/hy2/traffic/daily.json`，节点密码不保存到历史文件，只保存不可逆身份摘要；同名重建/端口复用、主节点密码轮换不会继承旧用户历史。
- 每分钟定时采集；查询时也执行采集。自动删除/到期节点的历史会在下次成功采集时清理，保留 30 个北京时间自然日。
- 采用**独立** `inet hy2_daily_traffic` nft 表，只统计该节点 UDP 端口双向字节，不修改 `hy2_pq` 或 `hy2_iplimit`。跨午夜分钟采样、断电尚未保存的尾部流量有少量归属/丢失风险，不能把它作为毫秒级结算系统。
- 本组件包含定时采集和关机 `ExecStop` 保存机制。历史测试机已有实际读写及正常重启保留记录；本轮新机结论单独记录在[验收](#acceptance)，CI 不能代替真机测试。

</details>

<a id="speed-logs"></a>

<details>
<summary>自动限速日志：全部端口、单端口、全部尚存历史</summary>

以下依次查询：最近 24 小时全部端口、最近 24 小时指定端口、指定端口全部尚存历史。北京时间输出；旧日志不代表当前配置，当前值以 `portbw report` 为准。

up / down 分别为上传 / 下载；没有自动降速日志不代表基础限速失效。可查历史受 journald 轮替、容量和保留策略限制。

```bash
TZ=Asia/Shanghai journalctl -u portbw-watch.service \
  --since '24 hours ago' --no-pager -o short-iso \
  | grep -E '自动降速|恢复基础速度|失败|Error|Traceback' \
  || echo '最近24小时没有匹配的降速、恢复或报错日志'
```

```bash
PORT=40002
TZ=Asia/Shanghai journalctl -u portbw-watch.service \
  --since '24 hours ago' --no-pager -o short-iso \
  | grep -E "${PORT} (up|down): (自动降速|恢复基础速度)" \
  || echo "最近24小时端口 ${PORT} 没有降速或恢复记录"
```

```bash
PORT=40002
TZ=Asia/Shanghai journalctl -u portbw-watch.service \
  --no-pager -o short-iso \
  | grep -E "${PORT} (up|down): (自动降速|恢复基础速度)" \
  || echo "端口 ${PORT} 没有尚存的降速或恢复历史记录"
```

</details>

<a id="hy2-audit"></a>

### HY2 来源 IP、连接错误与访问目标审计

**现已提供可执行的只读工具 `hy2-abuse.py`**，专门针对 HY2，而不是把 `vless-abuse` 改名。直接从受管理的 HY2 临时节点/IP 槽位，以及该节点的 `journalctl` 提取已有证据：

```bash
PORT=40020 DAYS=7 hy2-abuse
hy2-abuse 40020 --days 7 --json
```

- **当前来源 IP**：从 HY2 nftables `hy2_iplimit` 集合读取活跃槽位。若 `IP_LIMIT=0`，没有此类占位记录，**不能**据此认定“没有用户”。
- **异常事件**：从该 HY2 systemd 服务最多扫描最近 3000 条 journald 消息，按认证、握手、QUIC、超时、网络等类型统计警告/错误；不会在报告中输出原始错误字符串或节点密码。日志被轮替、事件未记录时无法补查。
- **实时访问目标**：如果手动配置了 Hysteria 2 **Traffic Stats API** 且仅绑定回环地址，可额外查询**当前 TCP 流**的目标地址、当前在线实例数（见安装说明）。**UDP 目标和历史目标列表不由该 API 提供**，不能虚构。
- **只读**：不触发自动封禁，也不更改配额、WG、nft 规则。IP 槽位/在线实例数不等于真实人数，出现异常事件也不等于已经确定滥用。

#### HY2 异常排查：按端口、日志类别与证据边界

审计工具只读取现有日志和运行状态，不安装历史域名采集代理，**不会自动识别或封禁滥用者**。应先核对日志和在线来源，再结合业务情况人工判断；未被记录的历史目标无法追溯。

| 想知道什么 | HY2 VPS 的查询方式 | 能得出的结论 |
|---|---|---|
| 某端口最近 7 天是否有认证/握手错误 | `hy2-abuse 40020 --days 7` | 在仍保留的 journald 样本中按类别统计错误/警告，不是全部连接记录 |
| 是否有特定来源 IP 正占用槽位 | `hy2-abuse 40020 --days 7 --json` | 只有启用 IP 槽位限制且对应集合存在，才有当前占位信息 |
| 主节点本身的服务日志 | `journalctl -u hy2.service --since '24 hours ago' --no-pager` | 可能含有敏感字段；**不应公开原始日志** |
| 当前 TCP 流量访问目标 | 先自行按 [Stats API 说明](#abuse-install)启用回环 API，再用 `--stats-url` | 当前 TCP 流，不能推导完整历史、UDP 目标或真实人数 |
| 是否另有整端口暂停/来源封禁 | `hy2-guard status` / `hy2-guard audit` | 手工防护规则，与访问日志不是同一个数据源 |

`--days 7` 的时间范围只约束 journal 查询；最多读取该服务最近 3000 条日志。超过范围、被 logrotate 清理或从未被记录的历史信息，不能通过修改天数追回。`auth`、`handshake`、`quic`、`timeout` 等类别代表事件类型，不代表某来源一定恶意；统计结果不能替代手工证据分析。

按端口定位临时服务及查看日志，直接使用[按端口查日志](#port-log)，无需手工查询 TAG。不要用 `ss -ltn` 的 TCP 监听结论判定 HY2 UDP/443 是否在线，也不要把 NAT 出口目标地址误当作用户来源。

<a id="socks5-daily"></a>

<details>
<summary>SOCKS5 每日流量查询：今天与最近 30 天</summary>

每日流量需先按[6.5](#socks5)开启采集；约每分钟采样，按北京时间保留最近 30 天，不等于实时配额余额：

```bash
socks5-traffic
socks5-traffic 41004
socks5-traffic 41004 --json
```

</details>

<a id="socks5-access"></a>

<details>
<summary>SOCKS5 使用审计／共享与滥用线索</summary>

访问审计按成功连接、原始主机名和各实例 journal 统计；日志通常在连接结束时写入，直接连接 IP 时不能推断域名，缺失或已清理的历史无法补查。天数范围 1～90：

```bash
socks5 access 41004 7
socks5 access-domains 41004 7
socks5 access-errors 41004 7
socks5 access-errors 41004 7 --limit 200
socks5 access-errors 41004 7 --all
```

`access` 提供概览，`access-domains` 列出全部成功连接的原始域名，`access-errors` 统计非零错误并列出明细；`--limit` 只改变明细条数。当前 TCP 连接与历史成功记录含义不同，不能把连接次数、来源 IP 或错误码直接当作人数、网页访问次数或恶意行为。

**SOCKS5 查询字段速查（所有命令仅在独立落地机执行）：**

| 需求 | 命令 | 特别注意 |
|---|---|---|
| 按端口看链接 | `socks5 link 41004` | 可能包含凭据，不能转发公开群组 |
| 查询余额和到期 | `socks5 list` | `LIMIT / USED / LEFT` 是合计流量，不是单向限额 |
| 当前源 IP 占位 | `socks5 ip-show 41004` | IP 不是实际用户数；槽位超时释放不强制断开已有 TCP |
| 查看某端口精确配额 | `socks5 quota 41004` | 不要误认为是每日统计 |
| 当前服务和规则自检 | `socks5 status` / `socks5 audit 41004` | 不是访问域名历史查询 |
| 过去 7 天访问域名和异常连接 | `socks5 access 41004 7` | 来自已有代理日志；原始日志若缺失无法补查 |
| 列出全部已记录的成功访问域名 | `socks5 access-domains 41004 7` | 直接按 IP 连接时不能推导一个域名 |
| 查询错误码明细 | `socks5 access-errors 41004 7 --all` | 错误不等同于已经证实滥用 |

日志与凭据属于敏感信息，应留在落地机的 root 私有环境。业务运行期间不要为了补齐历史日志而重装、重启现有 SOCKS5 服务。

</details>

<a id="subscription"></a>

<details>

<summary>导出有效临时节点订阅（生成文件）</summary>

只导出有效且运行的临时节点到 `/root/hy2_temp_subscription.txt` 和 Base64 文件，不包含主节点。Base64 不是加密。

```bash
hy2_temp_sub.sh

```

</details>

<a id="create"></a>

## 二、新建节点

<a id="temp-create"></a>

### HY2 临时节点

在 **HY2 VPS** 执行。下面两种命令二选一：示例有效期 7 天，总配额 50 GiB，最多 3 个活跃来源 IP；WG-NAT 命令需先完成[WG-NAT 部署](#wg-nat)。

普通临时节点：

```bash
DAYS=7; IP_VERSION=4 IP_LIMIT=3 PQ_GIB=50 D=$((DAYS*24*60*60)) hy2_mktemp.sh
```

WG-NAT 临时节点：

```bash
DAYS=7; IP_VERSION=4 IP_LIMIT=3 PQ_GIB=50 D=$((DAYS*24*60*60)) hy2_mktemp_nat.sh
```

<details>
<summary>其他创建方式：分钟、小时、固定端口、IPv6</summary>

### 按分钟

普通临时节点：

```bash
MINUTES=30; IP_VERSION=4 IP_LIMIT=3 PQ_GIB=1 D=$((MINUTES*60)) hy2_mktemp.sh
```

WG-NAT 临时节点：

```bash
MINUTES=30; IP_VERSION=4 IP_LIMIT=3 PQ_GIB=1 D=$((MINUTES*60)) hy2_mktemp_nat.sh
```

### 按小时

普通临时节点：

```bash
HOURS=2; IP_VERSION=4 IP_LIMIT=3 PQ_GIB=1 D=$((HOURS*60*60)) hy2_mktemp.sh
```

WG-NAT 临时节点：

```bash
HOURS=2; IP_VERSION=4 IP_LIMIT=3 PQ_GIB=1 D=$((HOURS*60*60)) hy2_mktemp_nat.sh
```

### 固定端口

下面示例固定使用端口 `40000`，有效期为 1 小时。

普通临时节点：

```bash
HOURS=1; IP_VERSION=4 PORT_START=40000 PORT_END=40000 IP_LIMIT=3 PQ_GIB=1 D=$((HOURS*60*60)) hy2_mktemp.sh
```

WG-NAT 临时节点：

```bash
HOURS=1; IP_VERSION=4 PORT_START=40000 PORT_END=40000 IP_LIMIT=3 PQ_GIB=1 D=$((HOURS*60*60)) hy2_mktemp_nat.sh
```

固定端口时，`PORT_START` 和 `PORT_END` 必须设置为同一个端口。

### 创建 IPv6 入站节点

IPv6 只需改一个参数，按下面两步操作：

1. 在 `/etc/default/hy2-main` 中填写 `HY_IPV6_DOMAIN`，并确保该域名的 AAAA 记录指向当前 HY2 VPS。
2. 将上面任意创建命令中的 `IP_VERSION=4` 改成 `IP_VERSION=6`，其他参数不用改。

例如，创建 IPv6 临时节点：

```bash
HOURS=2; IP_VERSION=6 IP_LIMIT=3 PQ_GIB=1 D=$((HOURS*60*60)) hy2_mktemp.sh
```

WG-NAT 同样只需把 `IP_VERSION=4` 改成 `IP_VERSION=6`；客户端通过 IPv6 连接 HY2 VPS，出站仍是 NAT 机的公网 IPv4。

常用调整：

- 不限制来源 IP：改为 `IP_LIMIT=0`
- 不限制流量：删除 `PQ_GIB=...`
- 修改固定端口：同时修改 `PORT_START` 和 `PORT_END`

创建成功后会直接输出端口、到期时间和节点链接。

**创建后如何验收、如何选择类型：** 先用 `hy2_audit.sh` 查到新 TAG、真实端口、剩余 TTL 与配额/IP 限制，再从自己的兼容客户端连接；仅 `systemctl active`、UDP 监听或脚本输出链接并不能证明公网客户端可用。普通节点从 VPS 公网出站；NAT 节点必须先核实自己的 WireGuard Peer、握手、mark 路由与出口防泄漏，不能将其失败时自动改走 VPS 出口。

| 创建意图 | 使用命令 | 对创建结果的关键检查 |
|---|---|---|
| 普通 UDP/QUIC 临时节点 | `hy2_mktemp.sh` | 对外 UDP 端口、SNI/证书、密码、配额与 IP 槽位 |
| WG-NAT 临时节点 | `hy2_mktemp_nat.sh` | 以上条件，加上专属 Peer、WG 路由和真实 NAT 出口 |
| IPv6 入站 | 增加 `IP_VERSION=6` | VPS 有全局 IPv6、AAAA/SNI/防火墙与客户端 IPv6 均正确 |

禁止在已有真实业务端口上运行固定端口创建示例；`PORT_START=PORT_END` 只是指定端口，不是绕过端口冲突检测的办法。TTL 和 30 天配额周期是两个时间维度，修改配置不会自动延长节点到期时间。

</details>

<details>
<summary>创建参数、节点类型与自动重置规则</summary>

普通节点用 VPS 出口，WG-NAT 节点用 NAT 出口；`id` 可省略或填写英文 / 数字 / 点 / 下划线 / 连字符。

| 参数 | 含义 |
|---|---|
| `D` | 有效期秒数，由示例前面的分钟 / 小时 / 天数计算 |
| `PQ_GIB` | 上传+下载合计配额；不限量时省略，不能填 0 |
| `IP_LIMIT` | 活跃来源 IP 槽位数；0 为不限数量 |
| `IP_STICKY_SECONDS` | 闲置占位秒数，默认 120；持续访问会刷新 |
| `PORT_START` / `PORT_END` | 固定端口时两项填相同端口 |
| `IP_VERSION` | 入站协议族 4 / 6 |

创建时有配额且有效期严格超过 30 天，才每 30×24 小时自动重置；不是每月 1 日，重置也不延长有效期。

</details>

<a id="socks5-create"></a>

<details>
<summary>可选 SOCKS5：新建账号、固定端口与家宽公网映射</summary>

先在**独立落地机**按[6.5](#socks5)手动安装；HY2 安装器不自动安装此模块。以下按分钟、小时、天及指定端口分别给出可复制示例。示例只在**新建或明确选定的独立 SOCKS5 落地机**执行，不在 HY2 入口机执行。

#### 创建 30 分钟账号

来源 IP 上限 1、闲置占位 60 秒、双向合计 1 GiB 配额：

```bash
MINUTES=30; IP_LIMIT=1 IP_STICKY_SECONDS=60 PQ_GIB=1 D=$((MINUTES*60)) socks5 add
```

#### 创建 2 小时账号

来源 IP 上限 2、闲置占位 300 秒、合计 5 GiB 配额：

```bash
HOURS=2; IP_LIMIT=2 IP_STICKY_SECONDS=300 PQ_GIB=5 D=$((HOURS*3600)) socks5 add
```

#### 创建 7 天账号

来源 IP 上限 3、闲置占位 600 秒、合计 50 GiB 配额：

```bash
DAYS=7; IP_LIMIT=3 IP_STICKY_SECONDS=600 PQ_GIB=50 D=$((DAYS*86400)) socks5 add
```

#### 固定本机监听端口

创建监听 `41004` 的 1 小时账号；省略 `PORT` 则由安装器选择本机空闲端口：

```bash
HOURS=1; IP_LIMIT=1 IP_STICKY_SECONDS=60 PQ_GIB=1 PORT=41004 D=$((HOURS*3600)) socks5 add
```


`IP_LIMIT=0` 不限来源 IP 数；省略 `PQ_GIB` 不限总流量，省略 `PORT` 自动选空闲端口。持续活动刷新来源占位时间，超时释放槽位不等于强制断开已建立连接。

**参数详细说明：**

| 参数 | 含义 | 适用说明 |
|---|---|---|
| `D` | 账号有效期，单位秒 | 可以由 `MINUTES`、`HOURS`、`DAYS` 变量计算；变量本身不是安装器子命令 |
| `IP_LIMIT` | 同时占用的来源 IP 槽位上限 | `0` 表示不限；不等于 TCP 会话数量 |
| `IP_STICKY_SECONDS` | 某来源 IP 闲置后释放名额的时间 | 活跃来源会刷新占位，不能据此保证强制踢出连接 |
| `PQ_GIB` | 合计上传+下载流量额度，GiB | 不设表示无总配额；配额在账号级累计 |
| `PORT` | 本机监听 TCP 端口 | 管理、配额、独立限速均按本机端口操作 |
| `--host` / `--public-port` | 客户端使用的公网域名和映射端口 | 不会替你创建路由器端口映射，需自行配置 TCP 映射 |

创建前先核对本机端口未被现有账号或其他服务占用；删除账号前核对 `socks5 list`，避免误删真实用户。原版 SOCKS5 不支持 UDP ASSOCIATE；即使与 HY2 同一仓库，也不代表它自动具备 UDP 代理能力。

家宽路由器需配置 **TCP** 公网端口映射；例如外部 `51004` 映射到本机 `41004`：

```bash
HOURS=1; IP_LIMIT=1 IP_STICKY_SECONDS=60 PORT=41004 D=$((HOURS*3600)) socks5 add --host proxy.example.com --public-port 51004
```

域名与外部端口换成真实值；后续管理及限速使用**本机监听端口 41004**。原样复用的版本仅支持 TCP CONNECT，不支持 UDP ASSOCIATE / BIND；没有新增 UDP 参数或转换 HY2 协议。

</details>

<a id="modify"></a>
## 三、修改与重置

<a id="quota-ip"></a>

### 配额与来源 IP

`pq_add.sh` 是重新设置额度：会清零已用量与已有自动重置周期，**不是追加流量**。需要保留周期应使用[保留周期重置](#quota-keep-cycle)。下列命令都按 HY2 节点本机监听端口操作。

<details>
<summary>设置 / 取消配额、设置 / 取消来源 IP 数量限制</summary>

**设置双向合计 50 GiB（已用量清零）：**

```bash
pq_add.sh 40020 50
```

**取消配额，不删除节点：**

```bash
pq_del.sh 40020
```

**最多 2 个活跃来源 IP，闲置占位 300 秒：**

```bash
ip_set.sh 40020 2 300
```

**取消 IP 数量限制，不删除节点：**

```bash
ip_del.sh 40020
```

执行后用 `hy2_audit.sh`、`pq_audit.sh 40020` 核对。删除来源 IP 限制后，地址族隔离仍保留；闲置槽位释放也不保证立即踢掉已有连接。

</details>

<details>
<summary>精确输入 MiB 配额（参数使用 GiB）</summary>

| 想要的额度 | 输入的 GiB | 示例 |
|---|---|---|
| 50 MiB | `0.048828125` | `pq_add.sh 40020 0.048828125` |
| 100 MiB | `0.09765625` | `pq_add.sh 40020 0.09765625` |
| 500 MiB | `0.48828125` | `pq_add.sh 40020 0.48828125` |

`PQ_GIB`、`pq_add.sh` 均按 GiB 计算，1 GiB = 1024 MiB。若要保留原周期，请勿使用普通 `pq_add.sh` 重设。

</details>

<a id="quota-keep-cycle"></a>

### 手动重置指定临时节点流量并保留原自动重置日期

**已实现 `pq_keepcycle.sh`**：直接使用 HY2 现有的配额库、临时节点/配额双锁、nft 原子重建和元数据所有者检查。对**已有配额且尚未到期的临时节点**，将总配额重新设定、已用量清零，**保留原 30 天周期日期**；若有效期严格超过 30 天、但此前 `pq_add.sh` 清除了周期，可按节点创建时间恢复应有的周期锚点。短期节点不会新增周期。

**历史验收（2026-11-05，旧测试机）**：独立 31 天 HY2-NAT 临时节点已用 41.07KiB 时调用 `pq_keepcycle.sh`，配额 20.48MiB → 40.96MiB、已用清零、原 30 天重置时间（北京时间 2026-12-05 09:20:25）不变，`KEEP_CYCLE_METADATA_TEST=PASS`，主 HY2/WG 正常。实际跨 30 天自动触发仍未验证。

示例：把 UDP 40020 的配额重置为 **500 MiB**，保留或按创建日恢复原周期：

```bash
pq_audit.sh 40020
pq_keepcycle.sh 40020 0.48828125
pq_audit.sh 40020
```

如果你还没有重新运行新版 `hy2.sh` 管理器，也可**只下载独立命令**，避免重装 WG 或修改运行中的 HY2：

```bash
curl -fL https://raw.githubusercontent.com/liucong552-art/hy2jieshu/main/pq_keepcycle.sh -o /root/pq_keepcycle.sh
bash -n /root/pq_keepcycle.sh
bash /root/pq_keepcycle.sh 40020 0.48828125
```

执行会在 `/root/pq-keepcycle-端口-随机字符/` 下备份节点和配额元数据（含敏感信息，保持 root 私有），然后更新配额和 nft 规则。与 `pq_add.sh` 的关键区别是**不随手删除原自动重置日期**。必须确认端口正确，且确定需要将已用量清零；该命令不是增加余额。

**测试边界**：历史记录支持已用清零与周期锚点保留；实际跨 30 天触发不在该记录范围。新机与当前代码的验证状态见[本轮验收](#acceptance)。测试只能使用专用节点，正式业务端口上执行会清零已用量。

<a id="quota-set-current-remaining"></a>

### 临时修改本期剩余流量（下周期恢复原配额）

可使用本项目已安装的 `/usr/local/lib/hy2/common.sh` 与 `quota-lib.sh`，**一行设置当前周期的剩余额度**。无需下载额外脚本或重装 HY2。**这是 README 提供的临时 Bash 操作，不代表系统已安装 `pq_set_remaining.sh` 命令**；它会改写真实计费配额及 nft 规则，首次操作应先在专用测试节点验证。

适用范围：**已有有效 30 天自动重置周期、尚未到期、原始配额正常的 HY2 受管临时节点**。例如原始总额度 200 GiB，本期希望**从此刻起还可使用 85 GiB**，只修改最前面的 `PORT=40020` 与 `LEFT_GIB=85` 两个值，其他内容完整复制到该 HY2 VPS 的 **root Bash 终端**。如果是短期节点、没有周期的节点、已到下次重置时间附近（60 秒内）或配额状态异常，命令会拒绝修改。**务必先确认真实端口和账号归属。**

<details>
<summary>展开：一行直接设置本期剩余流量（只改 PORT 和 LEFT_GIB）</summary>

```bash
PORT=40020 LEFT_GIB=85 bash -c 'set -Eeuo pipefail; umask 077; [[ "$(id -u)" == "0" ]] || { echo "请以root运行" >&2; exit 1; }; [[ "${PORT:-}" =~ ^[0-9]{1,5}$ ]] && (( 10#$PORT >= 1 && 10#$PORT <= 65535 )) || { echo "端口无效" >&2; exit 1; }; [[ -n "${LEFT_GIB:-}" ]] || { echo "请填写LEFT_GIB" >&2; exit 1; }; source /usr/local/lib/hy2/common.sh; source /usr/local/lib/hy2/quota-lib.sh; WANT="$(hy2_parse_gib_to_bytes "$LEFT_GIB")" || hy2_die "剩余额度无效"; [[ "$WANT" =~ ^[0-9]+$ ]] && (( WANT > 0 )) || hy2_die "剩余额度必须大于0"; hy2_ensure_runtime_dirs; hy2_acquire_lock_fd 7 "${HY2_LOCK_DIR}/temp.lock" 120 "节点锁繁忙"; export HY2_TEMP_LOCK_HELD=1; hy2_pq_lock; T="$(hy2_temp_meta_by_port "$PORT")"; [[ -n "$T" && -f "$T" ]] || hy2_die "不是受管HY2临时节点"; TAG="$(hy2_meta_get "$T" TAG)"; Q="$(hy2_quota_meta_file "$PORT")"; [[ -f "$Q" ]] || hy2_die "没有配额元数据"; [[ "$(hy2_meta_get "$Q" PORT)" == "$PORT" && "$(hy2_meta_get "$Q" OWNER_KIND)" == "temp" && "$(hy2_meta_get "$Q" OWNER_TAG)" == "$TAG" ]] || hy2_die "配额归属不匹配"; ORIG="$(hy2_meta_get "$Q" ORIGINAL_LIMIT_BYTES)"; EXPECTED="$(hy2_meta_get "$T" PQ_LIMIT_BYTES)"; PENDING="$(hy2_meta_get "$Q" REBUILD_PENDING)"; [[ "$ORIG" =~ ^[0-9]+$ && "$ORIG" == "$EXPECTED" && "$PENDING" == "0" ]] || hy2_die "配额结构异常或待恢复"; (( WANT <= ORIG )) || hy2_die "剩余不能超过原始周期配额"; NEXT="$(hy2_meta_get "$Q" NEXT_RESET_EPOCH)"; INTERVAL="$(hy2_meta_get "$Q" RESET_INTERVAL_SECONDS)"; EXPIRE="$(hy2_meta_get "$Q" EXPIRE_EPOCH)"; DURATION="$(hy2_meta_get "$Q" DURATION_SECONDS)"; CREATED="$(hy2_meta_get "$Q" CREATED_EPOCH)"; LAST="$(hy2_meta_get "$Q" LAST_RESET_EPOCH)"; START="$(hy2_meta_get "$T" CREATE_EPOCH)"; for V in "$NEXT" "$INTERVAL" "$EXPIRE" "$DURATION" "$CREATED" "$LAST" "$START"; do [[ "$V" =~ ^[0-9]+$ ]] || hy2_die "周期元数据无效"; done; NOW="$(date +%s)"; (( INTERVAL == 2592000 && DURATION > INTERVAL && NEXT > NOW + 60 && EXPIRE > NOW && START > 0 && START <= NOW && EXPIRE == START + DURATION )) || hy2_die "没有有效的未来30天周期或节点已到期"; [[ "$(hy2_meta_get "$T" DURATION_SECONDS)" == "$DURATION" && "$(hy2_meta_get "$T" EXPIRE_EPOCH)" == "$EXPIRE" ]] || hy2_die "节点生命周期元数据不匹配"; IFS="|" read -r STATE TOTAL USED LEFT <<< "$(hy2_pq_snapshot_unlocked "$PORT")"; [[ "$STATE" == active || "$STATE" == exhausted ]] && [[ "$TOTAL" == "$ORIG" && "$USED" =~ ^[0-9]+$ && "$LEFT" =~ ^[0-9]+$ ]] || hy2_die "实时nft配额不健康"; BACKUP="$(mktemp -d "/root/hy2-set-left-${PORT}-XXXXXXXX")"; chmod 700 "$BACKUP"; cp -p -- "$Q" "$BACKUP/quota.env"; cp -p -- "$T" "$BACKUP/node.env"; echo "原状态备份: $BACKUP"; NEW_USED=$((ORIG-WANT)); if ! hy2_pq_write_meta "$PORT" "$ORIG" "$NEW_USED" "$WANT" temp "$TAG" "$DURATION" "$EXPIRE" "$NEXT" "$INTERVAL" "$CREATED" "$LAST" "$NOW" 1; then hy2_die "写入失败；保留私有备份，请检查状态"; fi; if ! hy2_pq_rebuild_port "$PORT" "$WANT"; then hy2_die "nft规则重建失败；保留备份和待恢复标志，请先排查，不要盲目重试"; fi; if ! hy2_meta_upsert "$Q" REBUILD_PENDING 0; then hy2_die "恢复标志清理失败；保留备份，请先排查"; fi; echo "已设置端口 $PORT 当前剩余 $LEFT_GIB GiB；原周期总额与下次重置时间不变"; TZ=Asia/Shanghai date -d "@$NEXT" "+下次重置（北京时间）：%F %T"; echo "请执行 pq_audit.sh $PORT 核对；此处已用量是计费值，可能包含人为虚拟用量"'
```

</details>

**执行结果**：原始周期总额仍为 200 GiB，本期计费已用调整为 `200 − 85 = 115 GiB`、本期剩余变为 85 GiB；**下一个正常自动重置周期仍恢复 200 GiB**，下次重置日期、节点过期时间、端口、密码、订阅链接均不修改。期间的新流量继续消耗这 85 GiB。流量统计里的计费 `USED` 可能包含人为设置的虚拟用量，**不等于真实消耗**，真实日流量历史应另外查询 `hy2_traffic`。

**处理步骤与失败边界：** 此命令先获取临时节点锁和配额锁，核验配额与节点归属，检查有效 30 天周期和 nft 实时状态，然后在 `/root/hy2-set-left-端口-随机字符/` 备份原配额及节点元数据（root 私有），更新原计费已用量并重建本端口的 UDP 配额规则。它不会修改其他端口、WG、NAT、SOCKS5 或 portbw，不安装服务。**但是它会真实修改指定用户的计费配额与 nft 规则，可能影响正在传输的数据**；同一周期重复执行会重新赋予指定剩余额度。

若配额元数据写入后 nft 重建失败，脚本会非零退出、保留私有备份和 `REBUILD_PENDING` 标志；现有重建函数尝试将该端口 fail-closed。**此时不能盲目重试，也不能擅自清除待恢复标志**，应先检查 `pq_audit.sh`、`hy2_doctor.sh --strict` 与对应规则，由有经验的操作者定位并恢复。终端显示成功后也要核对：

```bash
pq_audit.sh 40020
hy2_audit.sh
```

**验证状态：** 此一行操作已依据当前 `hy2.sh` 的配额元数据/锁/重建函数进行代码核查，并用隔离模拟数据完成正常调整、耗尽后恢复、超额/无效周期/异常状态拒绝、重建失败保留待恢复标志等检查；**没有在真实 Debian VPS 或真实客户节点上执行，因此尚不能称为真机验收通过。** 建议首次使用前在专用测试节点验证当前余额、随后使用、重启持久化以及下一周期恢复，生产节点应先有可恢复快照。

<a id="quota-reset-time"></a>

<details>

<summary>查询端口下次自动重置时间及倒计时（只读）</summary>

只需修改端口；此查询读取 `/var/lib/hy2/quota`，不会修改内核配额。

```bash
PORT=40020
META="/var/lib/hy2/quota/$PORT.env"
if [[ -f "$META" ]]; then
  NEXT=$(awk -F= '$1=="NEXT_RESET_EPOCH"{print $2; exit}' "$META")
  INT=$(awk -F= '$1=="RESET_INTERVAL_SECONDS"{print $2; exit}' "$META")
  if [[ "$NEXT" =~ ^[0-9]+$ && "$INT" =~ ^[0-9]+$ ]] && (( NEXT>0 && INT>0 )); then
    TZ=Asia/Shanghai date -d "@$NEXT" "+下次计划重置（北京时间）：%F %T"
    NOW=$(date +%s)
    (( NEXT>NOW )) && echo "剩余 $((NEXT-NOW)) 秒" || echo "已到预定时间，请检查 pq-reset.timer"
  else
    echo "该端口没有启用周期重置"
  fi
else
  echo "没有该端口配额配置"
fi

```

`pq-reset.timer` 开机约 15 分钟后首次检查，以后通常每小时检查一次。

**自动重置与手动修改的区别：** 对有效期严格超过 30 天且创建时有配额的临时节点，周期重置使用节点自己的锚点，通常每 30×24 小时计算一次，并不会在北京时间每月 1 日统一重置。重设额度可能导致 `pq_add.sh` 清除原周期，需按具体业务需求选择 `pq_keepcycle.sh`。此处的查询命令是只读的，`pq-save.timer` 保存累计用量，与自动重置定时器职责不同。

```bash
systemctl list-timers --all pq-reset.timer
journalctl -u pq-reset.service -n 50 --no-pager

```

</details>

<a id="portbw"></a>

### 自动限速标准：40→18 Mbps，持续 45 秒触发

在实际需要限速的服务器执行；示例端口为 `40000`。整行复制即可：

```bash
portbw auto set 40000 --up 40 --down 40 --trigger 100 --after 45s --auto-up 18 --auto-down 18 --hold 120s --cooldown 0s
```

<a id="portbw-batch"></a>

#### portbw 批量操作（每项一行）

在运行 HY2 节点的 VPS 的 root SSH 终端执行。使用本仓库配套安装器安装的 `portbw`。按需要展开一项，复制整行。

<details>
<summary>批量开启 HY2 NAT 自动限速（默认 60 秒触发）</summary>

扫描本机标记为 `LANDING=nat` 的 HY2 临时节点。只改下面命令开头的参数。`UP` / `DOWN` 是基础上传 / 下载限速，`AUTO_UP` / `AUTO_DOWN` 是触发后的限速，单位均为 Mbps。`TRIGGER` 是基础限速的百分比，`AFTER` 是连续触发时间，`HOLD` 是滚动保护时间，`COOLDOWN` 是冷却时间，时长须带单位（如 `60s`）。降速值须低于对应的基础限速。参数一致且规则健康时，重复执行保留当前计时。

```bash
UP=40 DOWN=40 TRIGGER=100 AFTER=60s AUTO_UP=18 AUTO_DOWN=18 HOLD=120s COOLDOWN=0s python3 -IB -c 'exec("import json\nimport os\nfrom pathlib import Path\nimport re\nimport runpy\nimport shutil\nimport subprocess\nimport sys\n\nPROGRAM = Path(\"/usr/local/lib/portbw/portbw.py\")\nCLI = Path(\"/usr/local/sbin/portbw\")\nCONFIG = Path(\"/etc/portbw/config.json\")\nif os.geteuid() != 0:\n    sys.exit(\"请在 VPS 的 root SSH 终端执行。\")\nos.umask(0o077)\nfor key in list(os.environ):\n    if key.startswith(\"PORTBW_\"):\n        del os.environ[key]\nos.environ[\"PATH\"] = \"/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin\"\n\n\ndef native(required):\n    if PROGRAM.is_symlink() or (CLI.is_symlink() and CLI.resolve() != PROGRAM):\n        raise ValueError(\"portbw 的安装路径与 jieshu 不一致。\")\n    if not PROGRAM.is_file():\n        raise ValueError(\"找不到 jieshu 正式版 portbw 程序。\")\n    module = runpy.run_path(str(PROGRAM))\n    if any(not callable(module.get(name)) for name in required.split()):\n        raise ValueError(\"已安装的 portbw 版本不支持本命令。\")\n    return module\n\n\ndef run(*args, capture=False):\n    return subprocess.run(args, check=True, text=True, capture_output=capture, timeout=150).stdout\n\n\ndef run_main():\n    try:\n        main()\n    except Exception as error:\n        print(f\"操作停止：{error}\", file=sys.stderr)\n        sys.exit(1)\n\n\ndef main():\n    from types import SimpleNamespace\n\n    ports = {}\n    protected = {22, 443}\n    main_meta = Path(\"/var/lib/hy2/main/main.env\")\n    if main_meta.exists():\n        main_ports = [line.partition(\"=\")[2] for line in main_meta.read_text(encoding=\"utf-8\").splitlines()\n                      if line.startswith(\"MAIN_PORT=\")]\n        if len(main_ports) != 1 or not main_ports[0].isdigit() or not 1 <= int(main_ports[0]) <= 65535:\n            raise ValueError(\"无法识别 HY2 主节点端口\")\n        protected.add(int(main_ports[0]))\n    connection = os.environ.get(\"SSH_CONNECTION\", \"\").split()\n    if len(connection) == 4 and connection[-1].isdigit():\n        protected.add(int(connection[-1]))\n    for path in sorted(Path(\"/var/lib/hy2/temp\").glob(\"hy2-temp-*.env\")):\n        if path.is_symlink() or not path.is_file():\n            raise ValueError(f\"异常节点文件 {path.name}\")\n        values = {}\n        for line in path.read_text(encoding=\"utf-8\").splitlines():\n            key, _, value = line.partition(\"=\")\n            if key in (\"PORT\", \"LANDING\"):\n                if key in values:\n                    raise ValueError(f\"{path.name} 的 {key} 重复\")\n                values[key] = value\n        value = values.get(\"PORT\", \"\")\n        if not re.fullmatch(r\"[1-9][0-9]{0,4}\", value) or int(value) > 65535:\n            raise ValueError(f\"{path.name} 端口格式无效\")\n        port = int(value)\n        landing = values.get(\"LANDING\", \"\")\n        if landing not in (\"\", \"local\", \"nat\") or (not landing and path.name.startswith((\"hy2-temp-nat-\", \"hy2-temp-sync-nat-\"))):\n            raise ValueError(f\"{path.name} 的出口类型不明确\")\n        if port in ports:\n            raise ValueError(f\"端口 {port} 对应多个节点\")\n        ports[port] = landing == \"nat\"\n    targets = sorted(port for port, nat in ports.items() if nat)\n    if not targets:\n        print(\"没有 LANDING=nat 的 HY2 节点。\")\n        return\n    if protected.intersection(targets):\n        raise ValueError(\"NAT 清单包含 SSH 或HY2 主节点的受保护端口\")\n    m = native(\"lock cfg records auto_options commit audit_one report_all\")\n    parameters = (\"UP\", \"DOWN\", \"TRIGGER\", \"AFTER\", \"AUTO_UP\", \"AUTO_DOWN\", \"HOLD\", \"COOLDOWN\")\n    rates = SimpleNamespace(**{key.lower(): os.environ[key] for key in parameters})\n    options = m[\"auto_options\"](rates)\n    with m[\"lock\"]():\n        cfg, records = m[\"cfg\"](), m[\"records\"]()\n        if not cfg[\"tc_enabled\"]:\n            raise ValueError(\"自动限速需要启用 tc 的 portbw\")\n        for port in targets:\n            current = records.get(port)\n            if current and current.get(\"auto\") == options and m[\"audit_one\"](port, current, cfg)[0] == \"OK\":\n                print(f\"{port} 配置已一致，保留当前计时。\")\n                continue\n            m[\"commit\"](port, options[\"base\"][\"up\"], options[\"base\"][\"down\"], cfg, auto=options)\n            print(f\"{port} 已设为基础上传/下载 {rates.up}/{rates.down} Mbps，降速后 {rates.auto_up}/{rates.auto_down} Mbps。\")\n        m[\"report_all\"](cfg)\n\n\nrun_main()\n")'
```

</details>

<details>
<summary>解除本机全部 portbw 限速</summary>

解除本机所有由 `portbw` 管理的自动和静态限速，包括 NAT、普通节点和手动添加的端口。`portbw` 服务和节点继续运行。

```bash
python3 -IB -c 'exec("import json\nimport os\nfrom pathlib import Path\nimport re\nimport runpy\nimport shutil\nimport subprocess\nimport sys\n\nPROGRAM = Path(\"/usr/local/lib/portbw/portbw.py\")\nCLI = Path(\"/usr/local/sbin/portbw\")\nCONFIG = Path(\"/etc/portbw/config.json\")\nif os.geteuid() != 0:\n    sys.exit(\"请在 VPS 的 root SSH 终端执行。\")\nos.umask(0o077)\nfor key in list(os.environ):\n    if key.startswith(\"PORTBW_\"):\n        del os.environ[key]\nos.environ[\"PATH\"] = \"/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin\"\n\n\ndef native(required):\n    if PROGRAM.is_symlink() or (CLI.is_symlink() and CLI.resolve() != PROGRAM):\n        raise ValueError(\"portbw 的安装路径与 jieshu 不一致。\")\n    if not PROGRAM.is_file():\n        raise ValueError(\"找不到 jieshu 正式版 portbw 程序。\")\n    module = runpy.run_path(str(PROGRAM))\n    if any(not callable(module.get(name)) for name in required.split()):\n        raise ValueError(\"已安装的 portbw 版本不支持本命令。\")\n    return module\n\n\ndef run(*args, capture=False):\n    return subprocess.run(args, check=True, text=True, capture_output=capture, timeout=150).stdout\n\n\ndef run_main():\n    try:\n        main()\n    except Exception as error:\n        print(f\"操作停止：{error}\", file=sys.stderr)\n        sys.exit(1)\n\n\ndef main():\n    m = native(\"lock cfg records commit report_all\")\n    with m[\"lock\"]():\n        cfg, records = m[\"cfg\"](), m[\"records\"]()\n        for port in sorted(records):\n            m[\"commit\"](port, 0, 0, cfg, deleting=True)\n            print(f\"{port} 的 portbw 限速已解除。\")\n        if m[\"records\"]():\n            raise ValueError(\"仍有端口策略，未完成解除\")\n        m[\"report_all\"](cfg)\n    print(f\"已解除全部 {len(records)} 个端口的 portbw 限速。\")\n\n\nrun_main()\n")'
```

</details>

<details>
<summary>卸载 portbw 并清理专属残留</summary>

解除本机所有 `portbw` 限速，卸载其服务、程序、配置和状态，并清理专属 nftables 和 tc 资源。检测到其他程序的 tc 过滤器时保留共享 `clsact`；若专属 nft 表中有归属不明的规则，命令会停止。

```bash
python3 -IB -c 'exec("import json\nimport os\nfrom pathlib import Path\nimport re\nimport runpy\nimport shutil\nimport subprocess\nimport sys\n\nPROGRAM = Path(\"/usr/local/lib/portbw/portbw.py\")\nCLI = Path(\"/usr/local/sbin/portbw\")\nCONFIG = Path(\"/etc/portbw/config.json\")\nif os.geteuid() != 0:\n    sys.exit(\"请在 VPS 的 root SSH 终端执行。\")\nos.umask(0o077)\nfor key in list(os.environ):\n    if key.startswith(\"PORTBW_\"):\n        del os.environ[key]\nos.environ[\"PATH\"] = \"/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin\"\n\n\ndef native(required):\n    if PROGRAM.is_symlink() or (CLI.is_symlink() and CLI.resolve() != PROGRAM):\n        raise ValueError(\"portbw 的安装路径与 jieshu 不一致。\")\n    if not PROGRAM.is_file():\n        raise ValueError(\"找不到 jieshu 正式版 portbw 程序。\")\n    module = runpy.run_path(str(PROGRAM))\n    if any(not callable(module.get(name)) for name in required.split()):\n        raise ValueError(\"已安装的 portbw 版本不支持本命令。\")\n    return module\n\n\ndef run(*args, capture=False):\n    return subprocess.run(args, check=True, text=True, capture_output=capture, timeout=150).stdout\n\n\ndef run_main():\n    try:\n        main()\n    except Exception as error:\n        print(f\"操作停止：{error}\", file=sys.stderr)\n        sys.exit(1)\n\n\nUNITS = (\"portbw-watch.timer\", \"portbw-watch.service\", \"portbw-restore.service\")\nUNIT_DIR = Path(\"/etc/systemd/system\")\nDIRECTORIES = tuple(map(Path, (\"/usr/local/lib/portbw\", \"/etc/portbw\", \"/var/lib/portbw\",\n                              \"/var/lib/portbw-resident-release\", \"/run/portbw\")))\n\n\ndef state(unit):\n    text = run(\"systemctl\", \"show\", unit, \"-p\", \"LoadState\", \"-p\", \"ActiveState\",\n               \"-p\", \"FragmentPath\", \"-p\", \"DropInPaths\", capture=True)\n    values = dict(line.split(\"=\", 1) for line in text.splitlines() if \"=\" in line)\n    expected = UNIT_DIR / unit\n    if values[\"FragmentPath\"] not in (\"\", str(expected)) or values[\"DropInPaths\"]:\n        raise ValueError(f\"{unit} 存在其他安装位置或自定义配置，请先核对\")\n    if values[\"LoadState\"] not in (\"loaded\", \"not-found\"):\n        raise ValueError(f\"{unit} 状态异常：{values['"'"'LoadState'"'"']}\")\n    return values\n\n\ndef nft_table():\n    rows = json.loads(run(\"nft\", \"-j\", \"list\", \"tables\", capture=True))[\"nftables\"]\n    present = any(r.get(\"table\", {}).get(\"family\") == \"inet\" and r[\"table\"].get(\"name\") == \"pbw_policy\" for r in rows)\n    if present:\n        return json.loads(run(\"nft\", \"-j\", \"-a\", \"list\", \"table\", \"inet\", \"pbw_policy\", capture=True))[\"nftables\"]\n    return None\n\n\ndef maybe_remove_clsact(cfg):\n    if not cfg or not cfg[\"tc_enabled\"]:\n        return\n    iface = cfg[\"iface\"]\n    rules = lambda *args: json.loads(run(\"tc\", \"-j\", *args, capture=True))\n    shared = [r for r in rules(\"qdisc\", \"show\", \"dev\", iface) if r.get(\"kind\") == \"clsact\"]\n    if not shared:\n        return\n    filters = [rules(\"filter\", \"show\", \"dev\", iface, direction) for direction in (\"ingress\", \"egress\")]\n    if any(filters) or any(q.get(k) for q in shared for k in (\"ingress_block\", \"egress_block\")):\n        print(f\"{iface} 有其他或共享的 tc 规则，保留 clsact。\")\n        return\n    run(\"tc\", \"qdisc\", \"del\", \"dev\", iface, \"clsact\")\n\n\ndef main():\n    if any(p.is_symlink() or (p.exists() and not p.is_dir()) for p in DIRECTORIES):\n        raise ValueError(\"portbw 有目录类型异常，拒绝删除\")\n    units = {name: state(name) for name in UNITS}\n    loaded = [name for name, info in units.items() if info[\"LoadState\"] == \"loaded\"]\n    active = [name for name in loaded if units[name][\"ActiveState\"] in (\"active\", \"activating\", \"reloading\")]\n    table = nft_table()\n    if PROGRAM.exists() or PROGRAM.is_symlink() or CONFIG.exists() or table is not None or loaded:\n        m = native(\"lock cfg records commit _check_owned_table\")\n        if table is not None:\n            m[\"_check_owned_table\"](table, allow_missing=True)\n        cfg = m[\"cfg\"]() if CONFIG.exists() else None\n        records = m[\"records\"]()\n        if records and not cfg:\n            raise ValueError(\"存在策略但缺少配置，无法确定限速网卡\")\n        if cfg and cfg[\"tc_enabled\"]:\n            run(\"ip\", \"link\", \"show\", \"dev\", cfg[\"iface\"], capture=True)\n        try:\n            if loaded:\n                run(\"systemctl\", \"stop\", *loaded)\n            with m[\"lock\"]():\n                table = nft_table()\n                if table is not None:\n                    m[\"_check_owned_table\"](table, allow_missing=True)\n                for port in sorted(m[\"records\"]()):\n                    m[\"commit\"](port, 0, 0, cfg, deleting=True)\n                    print(f\"{port} 的 portbw 限速已解除。\")\n                table = nft_table()\n                if table is not None:\n                    m[\"_check_owned_table\"](table, allow_missing=True)\n                    run(\"nft\", \"delete\", \"table\", \"inet\", \"pbw_policy\")\n                maybe_remove_clsact(cfg)\n            if loaded:\n                run(\"systemctl\", \"disable\", *loaded)\n        except Exception:\n            if active:\n                try:\n                    run(\"systemctl\", \"start\", \"--no-block\", *active)\n                except Exception as restart:\n                    print(f\"恢复原监控服务也失败：{restart}\", file=sys.stderr)\n            raise\n    elif list(Path(\"/var/lib/portbw/ports\").glob(\"*.json\")):\n        raise ValueError(\"有保存的策略但找不到程序，请先恢复同版本 portbw\")\n    for path in (CLI, Path(\"/root/portbw.py\"), Path(\"/root/portbw-install.sh\"), *(UNIT_DIR / unit for unit in UNITS)):\n        path.unlink(missing_ok=True)\n    for path in DIRECTORIES:\n        if path.exists():\n            shutil.rmtree(path)\n    run(\"systemctl\", \"daemon-reload\")\n    if nft_table() is not None or any(state(unit)[\"LoadState\"] != \"not-found\" for unit in UNITS):\n        raise ValueError(\"有 portbw nft 表或服务残留\")\n    print(\"portbw 已卸载，专属程序、服务、配置和状态已清理。\")\n\n\nrun_main()\n")'
```

</details>

<details>
<summary>自动限速参数、恢复规则与关闭自动模式</summary>

上传 / 下载分别触发。100% 指本方向 40 Mbps；85% 只用于采样连续性判断，整段窗口仍需达标。保护至少 120 秒，高负载会续期；恢复后零冷却，再次触发仍需重新满足 45 秒条件。手动 set/up/down 会关闭自动模式。节点删除或到期会清理对应端口的限速记录，失败时保留节点元数据供重试。限速只覆盖本机端口，不覆盖任意 WG/NAT 转发；完整双向限速使用默认 nft+tc 模式。

同一端口的 TCP/UDP、IPv4/IPv6 共用上传额度，并共用另一份下载额度，不是每种协议各分一份。静态限速中 `0` 表示该方向不限速。`--nft-only` 不支持下载限速。

实际吞吐可能受协议开销、丢包和瞬时突发影响，不能要求客户端每一瞬间都精确等于配置值。

自动模式只有 `--cooldown` 可省略，默认 `60s`；零冷却必须明确填 `--cooldown 0s`，其他自动参数均必填。两方向均须满足 `0 < 降速值 < 基础速度`；`--trigger` 的单位为百分比，不是 Mbps。

设置后用 `portbw report` 核对；需要采样进度或内核故障检查时，见[补充查询](#advanced-queries)。

**自动降速完整核对顺序：** 设置前用 `portbw report` 保存原状态，设置后分别用 `portbw report`、`portbw auto status 40000` 和 `portbw audit` 查看端口基础限速、采样进度、异常状态。`portbw auto off 40000` 只取消自动策略、恢复该端口的基础上下行限制；`portbw del 40000` 则删除此端口整份 portbw 策略。两个动作不同，不应混用。端口上 TCP 与 UDP 上行共用一份聚合上限、下行另共用一份，**不是协议之间各分独立带宽**。

若服务日志发生轮转，`journalctl -u portbw-watch.service` 查询不到旧降速事件不等于此前从未发生过；也不能以单次吞吐低于设定 Mbps 断言守护进程故障。SOCKS5 的端口限速另在独立落地机操作其**本机 TCP 端口**，不在 HY2 VPS 替它设置。

```bash
portbw auto off 40000  # 关闭自动模式，恢复本端口的基础上传 / 下载限速
```

</details>

<details>
<summary>静态限速：设置、修改、取消</summary>

**40001：TCP+UDP 合计上传 10 Mbps / 合计下载 20 Mbps：**

```bash
portbw set 40001 10 20  # 40001：TCP+UDP 合计上传 10 Mbps / 合计下载 20 Mbps
```

**上传上限改为 15 Mbps：**

```bash
portbw up 40001 15      # 上传上限改为 15 Mbps
```

**下载上限改为 30 Mbps（直接限速，不是触发阈值）：**

```bash
portbw down 40001 30    # 下载上限改为 30 Mbps（直接限速，不是触发阈值）
```

**仅关闭下载限速；上传额度不变：**

```bash
portbw down 40001 0     # 仅关闭下载限速；上传额度不变
```

**仅取消端口限速，不删除 HY2 节点：**

```bash
portbw del 40001        # 仅取消端口限速，不删除 HY2 节点
```

</details>



**HY2-NAT 真机测速（2026-11-04）：** `portbw set 40020 10 10`、`portbw report`、`portbw audit` 均正常，客户端测速通过，最后 `portbw del 40020` 正常取消。**这只是历史测试机上特定链路的结果；自动降速与恢复的证据范围见[验收记录](#acceptance)，不等于当前 VPS 已完成同等高负载测试。**

**节点回收：** HY2 临时节点到期或删除时，会调用共享 `portbw del`，只删除该节点端口已保存的策略。发现对应记录但 CLI 缺失或删除失败时，保留节点元数据并报告失败，修复后重试；不清理其他端口、不卸载共享 portbw。WG-NAT 使用 HY2 VPS 上节点入口的端口策略，同数字端口的 TCP + UDP 共用额度；不要把 NAT 出口机的 WG `51820` 当作单个 HY2 节点限速端口。

<details>
<summary>可选 SOCKS5：配额、IP 槽位与主动恢复</summary>

仅在独立 SOCKS5 落地机执行：

```bash
socks5 ip-show 41004
socks5 ip-set 41004 2 60
socks5 ip-del 41004
socks5 pq-set 41004 5 --confirm-reset
socks5 pq-del 41004
socks5 watch
```

`pq-set` 将配额重设为 5 GiB、已用量清零并重设 30 天周期，需要显式 `--confirm-reset`；不是追加余额。`ip-set` 重置已有占位。`watch` 会修复规则或异常服务，不是只读查询。

| 管理动作 | 命令 | 影响范围 |
|---|---|---|
| 查看 IP 占位 | `socks5 ip-show 41004` | 查询，不修改占位 |
| 重设来源上限与占位 | `socks5 ip-set 41004 2 60` | 重置本账号的来源占位，可能影响正在使用的来源 |
| 取消来源上限 | `socks5 ip-del 41004` | 不删除账号 |
| 重设总配额为 5 GiB | `socks5 pq-set 41004 5 --confirm-reset` | **清空已有用量**；不是购买额外 5 GiB |
| 取消总配额 | `socks5 pq-del 41004` | 取消限制，风险应由操作者评估 |
| 主动修复防护/服务 | `socks5 watch` | **可能修改规则或服务状态**，不建议随意在生产落地机运行 |

涉及真实用户账号时，先记录原配额、到期时间、IP 占位和快照；不要把只读查询与会重置用量的命令混在一次批处理里。

SOCKS5 使用同一独立 [portbw](#portbw)，在落地机安装后对**本机**端口设置。原样 SOCKS5 的账号创建、到期、删除不自动增删 portbw 策略；HY2 的节点清理集成不作用于 SOCKS5。路由器映射 `51004 → 41004` 时，限速设置在 `41004`。

</details>

<a id="troubleshooting"></a>
## 四、故障排查

<a id="trouble-guide"></a>

### 出问题时，从现象找入口

| 现象 | 先检查什么 | 后续动作 |
|---|---|---|
| `command not found` | 对应管理组件是否已安装、是否仅下载未运行安装 | 按[安装](#install)补齐模块，不重装全部节点 |
| 不清楚当前核心版本 | `hysteria version \| grep '^Version:'` | 此命令只读，无需升级核心 |
| 服务 active、但外部客户端不通 | A 记录、正式 TLS/SNI、云防火墙 UDP/443、配额、IP 槽位 | 从另一设备验证；先查服务日志和 UDP 监听 |
| 只有某临时节点无法连接 | `hy2_audit.sh`、`pq_audit.sh 40020`、`hy2-guard status` | 先确认到期、来源限制、余额与黑名单 |
| 配额到期却未重置 | [按端口查下一次重置](#quota-reset-time)、`pq-reset.timer` | 再查服务日志，不立即清零计费 |
| 没有历史目标域名/UDP 目标 | `hy2-abuse` 的数据来源和 journald 保留范围 | HY2 审计无法补记不存在的历史 |
| Mbps 限速不等于实际下载速度 | `portbw report`、`portbw auto status 40020` | 排查丢包、拥塞、单位与实际方向 |
| 删除节点后仍显示规则 | `hy2_audit.sh`、`portbw report`、`hy2-guard status` | 检查清理是否返回错误，不能直接清空 nft |
| 重启后 WG-NAT 暂不可用 | `wg show wg-nat`、健康检查与握手 | 检查 fail-closed，避免回落到 VPS IP |
| SOCKS5 账号异常 | **落地机** `socks5 list`、`socks5 status` | 不应重装 HY2 |

<details>
<summary>服务、节点日志、每日流量、域名和 WG-NAT 的排查命令</summary>

#### 主节点与核心版本（HY2 VPS）

```bash
systemctl status hy2.service --no-pager
TZ=Asia/Shanghai journalctl -u hy2.service -n 120 --no-pager
hysteria version | grep '^Version:'
```

#### 临时节点、配额和管理规则

```bash
hy2_audit.sh
pq_audit.sh 40020
hy2-guard status
```

特定端口的日志按[按端口查日志](#port-log)操作；无须先找到 TAG。只有确定要恢复管理规则时，才考虑会修改运行状态的 `hy2_restore_all.sh`。

#### 每日流量采集

```bash
systemctl status hy2-traffic.timer hy2-traffic-shutdown.service --no-pager
TZ=Asia/Shanghai journalctl -u hy2-traffic.service -n 50 --no-pager
```

#### 域名解析、地址和 UDP 监听

```bash
getent ahostsv4 hy2.example.com
getent ahostsv6 hy2.example.com
ip -6 addr show scope global
ss -lun '( sport = :443 )'
```

域名和 `443` 是示例值，修改了主节点的 `HY_LISTEN` 时改为实际 UDP 端口；首次签证书还需检查 TCP/80。

#### WG-NAT（只在已经配置 NAT 的 HY2 VPS）

```bash
wg show wg-nat
/usr/local/sbin/wg_nat_healthcheck.sh
```

NAT 端进一步排查时才在独立出口机执行 `bash /root/nat.sh status`。检查不应通过重新初始化 WG、增删生产 Peer 或关闭防泄漏来试错。

</details>

<a id="port-log"></a>

<details>
<summary>按端口查询 HY2 临时节点状态及最近 100 条日志（只改 PORT）</summary>

在 HY2 VPS 的 root Bash 执行，**只需改最前面的 `PORT=40020`**；无需知道节点 TAG。查询通过受管节点元数据定位唯一的临时服务，不会重启或删除节点。日志可能含敏感信息，公开分享前先清除地址、凭据等内容。

```bash
PORT=40020 bash -c 'set -Eeuo pipefail; [[ "$PORT" =~ ^[1-9][0-9]{0,4}$ ]] && (( 10#$PORT <= 65535 )) || { echo "端口无效" >&2; exit 1; }; source /usr/local/lib/hy2/common.sh; META="$(hy2_temp_meta_by_port "$PORT")"; [[ -n "$META" && -f "$META" ]] || hy2_die "不是受管HY2临时节点"; TAG="$(hy2_meta_get "$META" TAG)"; /usr/local/sbin/hy2_audit.sh --tag "$TAG"; TZ=Asia/Shanghai journalctl -u "${TAG}.service" -n 100 --no-pager'
```

主节点日志另用 `journalctl -u hy2.service -n 100 --no-pager` 查询；主节点不能按临时节点方式删除。如果是配额异常，先运行 `pq_audit.sh 40020`，不要直接把日志错误理解为配额耗尽。

</details>

<a id="service-management"></a>

<details>
<summary>服务管理、重启恢复与证书检查</summary>

主节点与每个临时节点各自为一个 systemd 服务；管理器安装后启用恢复、配额保存、到期回收和看门狗。每日流量与手工黑名单安装后各有独立服务。

| 服务 / 定时器 | 作用 |
|---|---|
| `hy2.service` | 主节点；`systemctl restart hy2.service` 只重启主节点 |
| `hy2-temp-tmp-节点名称.service` / `hy2-temp-nat-节点名称.service` | 新建的普通 / WG-NAT 临时节点 |
| `hy2-managed-restore.service` | 开机恢复受管理配额、IP 限制和节点保护 |
| `hy2-managed-watchdog.timer` | 定期检查并恢复异常临时节点，失败保持阻断 |
| `hy2-gc.timer` | 回收自然到期临时节点 |
| `pq-save.timer` / `pq-reset.timer` | 定期保存已用量 / 检查到期的 30 天周期 |
| `hy2-managed-shutdown-save.service` | 正常关机前保存受管理配额 |

`hy2_audit.sh` 按实际出口显示普通节点的 `hy2-temp-tmp-` 和 NAT 节点的 `hy2-temp-nat-` 前缀。名称末尾带 `*` 的旧节点保留原来的实际服务名，管理时可按端口查找。新节点的分享名称与服务名去掉 `.service` 后一致。

```bash
systemctl list-timers --all hy2-managed-watchdog.timer hy2-gc.timer pq-save.timer pq-reset.timer
systemctl status hy2-managed-restore.service hy2-managed-shutdown-save.service --no-pager
```

临时停止服务后，看门狗可能重新启动它；需要暂停用户连接时用 `hy2-guard pause 端口`，永久撤销用[删除节点](#node-manage)。正常 `reboot` 会经过保存流程；异常断电可能丢失最后一次保存后的流量，不将这两类情况混为一谈。

证书通过 acme.sh 定时续期及 HY2 reload hook 验证和应用；先检查证书与日志，避免为排障反复强制签发而触发 CA 限额：

```bash
CERT=$(awk -F= '$1=="TLS_CERT"{print substr($0,index($0,"=")+1); exit}' /var/lib/hy2/main/main.env)
openssl x509 -in "$CERT" -noout -subject -issuer -dates
journalctl -u hy2.service -n 50 --no-pager
```

证书路径由 `/var/lib/hy2/main/main.env` 的 `TLS_CERT` 读取，通常为 `/etc/hysteria/certs/域名/fullchain.pem`。续期测试、升级回滚和故障注入的实际验收范围见[验收记录](#acceptance)。

</details>

<a id="remove"></a>
## 五、暂停、封禁与删除

<a id="port-block"></a>

两类防护互不替代：整端口暂停阻止该端口的外部连接；来源黑名单只阻止指定来源访问该端口。两者都覆盖同一数字端口的 TCP/UDP，支持 IPv4/IPv6，并由本项目的独立管理服务保存和恢复。

| 目标 | 使用入口 | 修改范围 |
|---|---|---|
| 暂停整个端口 | `hy2-guard pause 40020` | 不删除节点、不清除配额 |
| 按来源 IP 或网段封禁 | `hy2-guard ban 40020 203.0.113.0/24` | 不影响其他端口 |
| 查询已管理的暂停与黑名单 | `hy2-guard status` | 只读 |
| 仅允许指定来源访问本机端口 | [来源 IP 白名单](#source-allowlist) | IPv4 / IPv6，可选 TCP / UDP |

<details>
<summary>首次安装手工防护工具（HY2 VPS）</summary>

```bash
curl -fL --retry 3 \
  'https://raw.githubusercontent.com/liucong552-art/hy2jieshu/main/hy2-guard.py' \
  -o /root/hy2-guard.py &&
python3 -m py_compile /root/hy2-guard.py &&
python3 /root/hy2-guard.py install &&
hy2-guard audit
```

本模块独立管理 `inet hy2_manual_guard` 表，不清空 nftables 全局规则。请勿把 SSH 管理端口误用于 HY2 节点，以免被同步阻断。

</details>

<a id="pause-port"></a>

### 端口暂停：封禁、解封、状态查询

<details>
<summary>展开端口暂停命令及适用范围（只改端口）</summary>

**1. 暂停整个端口：**

```bash
hy2-guard pause 40020
```

**2. 恢复整个端口：**

```bash
hy2-guard resume 40020
```

**3. 查询所有受管理暂停及黑名单：**

```bash
hy2-guard status
hy2-guard audit
```

暂停并不停止 Hysteria 服务，恢复也不会撤销该端口原有的来源黑名单、配额、IP 数量限制和 `portbw` 限速。不要通过停止 systemd 临时服务代替暂停，看门狗可能重新启动它。

</details>

<a id="source-blacklist"></a>

### 来源 IP / CIDR 黑名单

在 **HY2 VPS** 以 root 执行，目标为本机 HY2 监听端口。支持 IPv4、IPv6 和规范 CIDR，例如 `203.0.113.0/24`（不可写成 `203.0.113.123/24`）。示例 IP 均为文档保留地址，执行时换成需要处理的真实来源。

#### 1. 永久拉黑单 IP / 网段

<details>
<summary>展开封禁命令（支持逐条或批量）</summary>

```bash
hy2-guard ban 40020 198.51.100.8
hy2-guard ban 40020 203.0.113.0/24
hy2-guard ban 40020 2001:db8::/32
```

按端口批量一行执行（只改 `P` 和 `IPS`）：

```bash
P=40020 IPS="198.51.100.8 203.0.113.0/24" bash -c 'set -Eeuo pipefail; for ip in $IPS; do hy2-guard ban "$P" "$ip"; done; hy2-guard audit'
```

此循环不是跨多条规则的原子事务：其中一项失败时，之前成功添加的记录仍可能生效。必须查 `hy2-guard status` 核对。

</details>

<a id="blacklist-report"></a>

#### 2. 一键查询全部黑名单（只读）

```bash
hy2-guard status
hy2-guard audit
```

#### 3. 解除指定 IP 或网段

<details>
<summary>展开逐条及批量解封命令</summary>

```bash
hy2-guard unban 40020 198.51.100.8
hy2-guard unban 40020 203.0.113.0/24
```

```bash
P=40020 IPS="198.51.100.8 203.0.113.0/24" bash -c 'set -Eeuo pipefail; for ip in $IPS; do hy2-guard unban "$P" "$ip"; done; hy2-guard audit'
```

`unban` 只解除匹配的来源封禁，整端口 `pause` 不变。

</details>

#### 4. 清空指定端口所有来源黑名单

```bash
hy2-guard clear 40020
```

只清理该端口的来源黑名单，**不撤销整端口暂停**。要取消暂停使用 `hy2-guard resume 40020`。

#### 5. 验证与注意事项

```bash
hy2-guard status
hy2-guard audit
systemctl is-active hy2-manual-guard.service
```

不应手工清除整张 nftables 规则表；节点成功删除时会清理其受管暂停和来源黑名单，防止端口复用继承旧规则。`IP_LIMIT` 是来源 IP 占位数控制，与永久黑名单不是同一件事。

<a id="source-allowlist"></a>

### 来源 IP 白名单

按来源 IPv4/IPv6 限制本机监听端口的 TCP 或 UDP 入站流量。在实际监听该端口的主机上，用 **root + Bash** 逐条执行。需要 `nftables`、Python 3.9 或更新版本，以及 systemd。`P` 填本机实际监听端口，IP 填主机看到的客户端来源地址。

| `PROTO` | 应用范围 |
|---|---|
| `tcp` | TCP 入口，例如 VLESS、SOCKS5 |
| `udp` | UDP 入口，例如 Hysteria2 |
| `both` | 同一数字端口的 TCP 和 UDP 共用一份 IP 名单 |

选定协议后，`IPS4` 或 `IPS6` 留空会拦截该地址族的非回环入站流量，至少填写一个地址。名单内的地址仍需满足主机上其他防火墙规则。

<details>
<summary>展开四条单行命令</summary>

#### ① 添加或修改白名单

修改行首的 `P`、`PROTO`、`IPS4` 和 `IPS6`。把示例 IP 换成实际允许的客户端出口 IP，多个地址用空格分隔。再次执行会替换该端口的协议选择和 IP 名单，并保存配置、启用开机恢复。

```bash
P=41004 PROTO=both IPS4="203.0.113.10 198.51.100.20" IPS6="" python3 -c 'exec("import fcntl\nimport ipaddress\nimport os\nimport subprocess\nfrom pathlib import Path\n\nport = int(os.environ[\"P\"])\nif os.geteuid() != 0 or not 1 <= port <= 65535:\n    raise SystemExit(\"需要 root，端口必须为 1–65535\")\nmode = os.environ.get(\"PROTO\", \"tcp\").lower()\nif mode not in (\"tcp\", \"udp\", \"both\"):\n    raise SystemExit(\"PROTO 必须为 tcp、udp 或 both\")\nprotocols = (\"tcp\", \"udp\") if mode == \"both\" else (mode,)\n\n\ndef addresses(name, address_type):\n    values = os.environ[name].replace(\",\", \" \").split()\n    if any(\"%\" in ip for ip in values):\n        raise SystemExit(\"请填写不带网卡后缀的 IP\")\n    return list(dict.fromkeys(str(address_type(ip)) for ip in values))\n\n\nipv4 = addresses(\"IPS4\", ipaddress.IPv4Address)\nipv6 = addresses(\"IPS6\", ipaddress.IPv6Address)\nif not (ipv4 or ipv6):\n    raise SystemExit(\"IPv4、IPv6 不能同时为空\")\n\ntable = f\"s5wl_{port}\"\ndirectory = Path(\"/etc/jsh-whitelist\")\nunit = Path(\"/etc/systemd/system/jsh-whitelist@.service\")\nhead = (\n    f\"add table inet {table}\\n\"\n    f\"add chain inet {table} input {{ type filter hook input priority -30; policy accept; }}\\n\"\n)\nrules = \"\"\nfor protocol in protocols:\n    for family, source, ips in ((\"ipv4\", \"ip\", ipv4), (\"ipv6\", \"ip6\", ipv6)):\n        exception = f\"{source} saddr != {{ {\u0027, \u0027.join(ips)} }} \" if ips else \"\"\n        rules += (\n            f\u0027add rule inet {table} input meta nfproto {family} \u0027\n            f\u0027iifname != \"lo\" {protocol} dport {port} {exception}drop\\n\u0027\n        )\n\nwith open(\"/run/lock/jsh-whitelist.lock\", \"a\") as lock:\n    fcntl.flock(lock, fcntl.LOCK_EX)\n    tables = subprocess.check_output([\"nft\", \"-n\", \"list\", \"tables\"], text=True)\n    exists = f\"table inet {table}\" in tables.splitlines()\n    batch = (f\"flush chain inet {table} input\\n\" if exists else head) + rules\n    subprocess.run([\"nft\", \"-c\", \"-f\", \"-\"], input=batch, text=True, check=True)\n    subprocess.run([\"nft\", \"-f\", \"-\"], input=batch, text=True, check=True)\n    directory.mkdir(mode=0o700, exist_ok=True)\n    temp = directory / f\".{port}.tmp\"\n    temp.write_text(head + rules, encoding=\"utf-8\")\n    temp.chmod(0o600)\n    temp.replace(directory / f\"{port}.nft\")\n    unit.write_text(\"\"\"[Unit]\nDescription=Jieshu IP whitelist %i\nDefaultDependencies=no\nAfter=local-fs.target nftables.service\nBefore=sysinit.target network-pre.target\n[Service]\nType=oneshot\nRemainAfterExit=yes\nExecStartPre=-/usr/sbin/nft delete table inet s5wl_%i\nExecStart=/usr/sbin/nft -f /etc/jsh-whitelist/%i.nft\n[Install]\nWantedBy=sysinit.target\n\"\"\", encoding=\"utf-8\")\n    subprocess.run([\"systemctl\", \"daemon-reload\"], check=True)\n    subprocess.run([\"systemctl\", \"enable\", f\"jsh-whitelist@{port}.service\"], check=True)\n\nprint(f\"端口 {port}：白名单已更新，开机恢复已启用\")\nprint(\"协议：\" + \"、\".join(protocol.upper() for protocol in protocols))\nprint(\"IPv4：\" + (\"、\".join(ipv4) or \"禁止\"))\nprint(\"IPv6：\" + (\"、\".join(ipv6) or \"禁止\"))")'
```

#### ② 查询全部白名单

直接执行，显示本机的白名单端口、协议、IPv4/IPv6 名单、运行状态和开机恢复状态。

```bash
python3 -c 'exec("import ipaddress\nimport re\nimport subprocess\nfrom pathlib import Path\n\ndirectory = Path(\"/etc/jsh-whitelist\")\ntables = subprocess.check_output([\"nft\", \"-n\", \"list\", \"tables\"], text=True)\nlive = {int(p) for p in re.findall(r\"(?m)^table inet s5wl_([1-9][0-9]{0,4})$\", tables)}\nsaved = {\n    int(file.stem): file for file in directory.glob(\"*.nft\")\n    if re.fullmatch(r\"[1-9][0-9]{0,4}\", file.stem)\n}\nports = sorted(p for p in live | saved.keys() if p <= 65535)\nfamilies = {\"2\": 4, \"ipv4\": 4, \"10\": 6, \"ipv6\": 6}\nsources = {\"ip\": 4, \"ip6\": 6}\n\n\ndef parse(lines, port):\n    rule = re.compile(\n        rf\u0027(?:meta nfproto (ipv4|ipv6|2|10)\\s+)?\u0027\n        rf\u0027iifname\\s+!=\\s+\"lo\"\\s+(tcp|udp)\\s+dport\\s+{port}\\s+\u0027\n        rf\u0027(?:(ip6?) saddr\\s+!=\\s+(\\{{[^{{}}]+\\}}|\\S+)\\s+)?drop\u0027\n    )\n    lists = {}\n    valid = bool(lines)\n    for line in lines:\n        match = rule.fullmatch(line.strip())\n        if not match:\n            valid = False\n            continue\n        family, protocol, source, values = match.groups()\n        explicit = families.get(family)\n        version = sources.get(source, explicit)\n        if version is None or explicit not in (None, version):\n            valid = False\n            continue\n        key = protocol, version\n        if key in lists:\n            valid = False\n            continue\n        try:\n            ips = [ipaddress.ip_address(ip.strip()) for ip in values.strip(\"{}\").split(\",\")] if values else []\n            if any(ip.version != version or \"%\" in str(ip) for ip in ips):\n                raise ValueError\n            lists[key] = \"、\".join(map(str, ips)) or \"禁止\"\n        except ValueError:\n            valid = False\n    protocols = [p for p in (\"tcp\", \"udp\") if any((p, v) in lists for v in (4, 6))]\n    valid = valid and bool(protocols) and all(\n        (p, v) in lists for p in protocols for v in (4, 6)\n    )\n    return protocols, lists, valid\n\n\nprint(f\"========== IP白名单（{len(ports)}个端口）==========\")\nfor port in ports:\n    table = f\"s5wl_{port}\"\n    if port in live:\n        result = subprocess.run(\n            [\"nft\", \"-n\", \"list\", \"chain\", \"inet\", table, \"input\"],\n            capture_output=True, text=True\n        )\n        lines = [line.strip() for line in result.stdout.splitlines() if line.strip()]\n        has_hook = any(re.fullmatch(r\"type filter hook input priority .+; policy accept;\", line) for line in lines)\n        headers = (f\"table inet {table} {{\", \"chain input {\", \"}\")\n        lines = [line for line in lines if line not in headers and not line.startswith(\"type \")]\n        if result.returncode or not has_hook:\n            lines = []\n    else:\n        prefix = f\"add rule inet {table} input \"\n        lines = [\n            line.removeprefix(prefix) for line in saved[port].read_text(encoding=\"utf-8\").splitlines()\n            if line.startswith(prefix)\n        ]\n    protocols, lists, valid = parse(lines, port)\n    state = (\"正常\" if valid else \"规则异常\") if port in live else \"未生效\"\n    enabled = port in saved and subprocess.run(\n        [\"systemctl\", \"is-enabled\", \"--quiet\", f\"jsh-whitelist@{port}.service\"],\n        stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL\n    ).returncode == 0\n    boot = \"已启用\" if enabled else \"未启用\"\n    print(f\"\\n【端口 {port}】 {state} | 开机恢复：{boot}\")\n    print(\"协议：\" + (\"、\".join(p.upper() for p in protocols) or \"规则异常\"))\n    for protocol in protocols:\n        for version in (4, 6):\n            print(f\"{protocol.upper()} IPv{version}：{lists.get((protocol, version), \u0027规则异常\u0027)}\")\n\nif not ports:\n    print(\"没有设置任何白名单\")")'
```

#### ③ 解除指定端口白名单

修改行首的 `P`，删除指定端口的白名单规则和保存文件，并禁用该端口的开机恢复。

```bash
P=41004 python3 -c 'exec("import fcntl\nimport os\nimport re\nimport subprocess\nimport sys\nfrom pathlib import Path\n\nif os.geteuid() != 0:\n    raise SystemExit(\"必须使用 root\")\nall_ports = sys.argv[1:] == [\"--all\"]\nport = None if all_ports else int(os.environ[\"P\"])\nif port is not None and not 1 <= port <= 65535:\n    raise SystemExit(\"端口必须为 1–65535\")\ndirectory = Path(\"/etc/jsh-whitelist\")\ntemplate = Path(\"/etc/systemd/system/jsh-whitelist@.service\")\nboot_directory = template.parent / \"sysinit.target.wants\"\n\n\ndef numbers(pattern, body):\n    return {int(item) for item in re.findall(pattern, body) if 1 <= int(item) <= 65535}\n\n\nwith open(\"/run/lock/jsh-whitelist.lock\", \"a\") as lock:\n    fcntl.flock(lock, fcntl.LOCK_EX)\n    tables = subprocess.check_output([\"nft\", \"-n\", \"list\", \"tables\"], text=True)\n    live = numbers(r\"(?m)^table inet s5wl_([1-9][0-9]{0,4})$\", tables)\n    listed = subprocess.check_output(\n        [\"systemctl\", \"list-unit-files\", \"--no-legend\", \"--no-pager\"], text=True\n    )\n    loaded = subprocess.check_output(\n        [\"systemctl\", \"list-units\", \"--type=service\", \"--all\", \"--plain\",\n         \"--no-legend\", \"--no-pager\"], text=True\n    )\n    links = \"\\n\".join(link.name for link in boot_directory.glob(\"jsh-whitelist@*.service\"))\n    units = numbers(\n        r\"(?m)^\\s*jsh-whitelist@([1-9][0-9]{0,4})\\.service(?:\\s|$)\",\n        \"\\n\".join((listed, loaded, links))\n    )\n    saved = numbers(\n        r\"(?m)^([1-9][0-9]{0,4})\\.nft$\", \"\\n\".join(file.name for file in directory.glob(\"*.nft\"))\n    )\n    targets = sorted(live | saved | units) if all_ports else [port]\n    for item in targets:\n        if item in units or template.is_file():\n            subprocess.run(\n                [\"systemctl\", \"disable\", \"--now\", f\"jsh-whitelist@{item}.service\"],\n                check=True\n            )\n        if item in live:\n            subprocess.run([\"nft\", \"delete\", \"table\", \"inet\", f\"s5wl_{item}\"], check=True)\n        (directory / f\"{item}.nft\").unlink(missing_ok=True)\n        (directory / f\".{item}.tmp\").unlink(missing_ok=True)\n        print(f\"端口 {item}：白名单已解除\")\n\nif all_ports:\n    print(f\"共解除 {len(targets)} 个端口\" if targets else \"没有需要解除的白名单\")")'
```

#### ④ 解除所有端口白名单

**直接执行会批量解除本机所有由本节命令管理的端口白名单**，包括运行规则、保存文件和开机启用项。

```bash
python3 -c 'exec("import fcntl\nimport os\nimport re\nimport subprocess\nimport sys\nfrom pathlib import Path\n\nif os.geteuid() != 0:\n    raise SystemExit(\"必须使用 root\")\nall_ports = sys.argv[1:] == [\"--all\"]\nport = None if all_ports else int(os.environ[\"P\"])\nif port is not None and not 1 <= port <= 65535:\n    raise SystemExit(\"端口必须为 1–65535\")\ndirectory = Path(\"/etc/jsh-whitelist\")\ntemplate = Path(\"/etc/systemd/system/jsh-whitelist@.service\")\nboot_directory = template.parent / \"sysinit.target.wants\"\n\n\ndef numbers(pattern, body):\n    return {int(item) for item in re.findall(pattern, body) if 1 <= int(item) <= 65535}\n\n\nwith open(\"/run/lock/jsh-whitelist.lock\", \"a\") as lock:\n    fcntl.flock(lock, fcntl.LOCK_EX)\n    tables = subprocess.check_output([\"nft\", \"-n\", \"list\", \"tables\"], text=True)\n    live = numbers(r\"(?m)^table inet s5wl_([1-9][0-9]{0,4})$\", tables)\n    listed = subprocess.check_output(\n        [\"systemctl\", \"list-unit-files\", \"--no-legend\", \"--no-pager\"], text=True\n    )\n    loaded = subprocess.check_output(\n        [\"systemctl\", \"list-units\", \"--type=service\", \"--all\", \"--plain\",\n         \"--no-legend\", \"--no-pager\"], text=True\n    )\n    links = \"\\n\".join(link.name for link in boot_directory.glob(\"jsh-whitelist@*.service\"))\n    units = numbers(\n        r\"(?m)^\\s*jsh-whitelist@([1-9][0-9]{0,4})\\.service(?:\\s|$)\",\n        \"\\n\".join((listed, loaded, links))\n    )\n    saved = numbers(\n        r\"(?m)^([1-9][0-9]{0,4})\\.nft$\", \"\\n\".join(file.name for file in directory.glob(\"*.nft\"))\n    )\n    targets = sorted(live | saved | units) if all_ports else [port]\n    for item in targets:\n        if item in units or template.is_file():\n            subprocess.run(\n                [\"systemctl\", \"disable\", \"--now\", f\"jsh-whitelist@{item}.service\"],\n                check=True\n            )\n        if item in live:\n            subprocess.run([\"nft\", \"delete\", \"table\", \"inet\", f\"s5wl_{item}\"], check=True)\n        (directory / f\"{item}.nft\").unlink(missing_ok=True)\n        (directory / f\".{item}.tmp\").unlink(missing_ok=True)\n        print(f\"端口 {item}：白名单已解除\")\n\nif all_ports:\n    print(f\"共解除 {len(targets)} 个端口\" if targets else \"没有需要解除的白名单\")")' --all
```

</details>

**实测结果**　在 Debian 13 VPS 的临时监听端口验证了 IPv4/IPv6 的 TCP、UDP 放行与拦截，以及查询、单端口解除和全部解除。实际重启 VPS 后，systemd 自动加载了白名单，放行与拦截复测通过。

<a id="node-manage"></a>

<details>
<summary>按端口强制删除 HY2 临时节点 / 清空全部临时节点</summary>

### 按端口强制删除临时节点

**在 HY2 VPS 以 root 身份执行，直接按端口删除，不需要查 TAG。** 只修改最前面的 `PORT=40020` 为实际临时节点端口。删除会立即断开该节点连接并使链接失效，清理对应的配额、来源 IP 限制、手工暂停/黑名单及已保存的 portbw 端口策略；不会删除主节点或其他端口节点。执行前先用 `hy2_audit.sh` 确认端口归属。

```bash
PORT=40020 bash -c 'set -Eeuo pipefail; [[ "$(id -u)" == 0 ]] || { echo "需要 root" >&2; exit 1; }; [[ "$PORT" =~ ^[1-9][0-9]{0,4}$ ]] && (( 10#$PORT <= 65535 )) || { echo "端口无效" >&2; exit 1; }; source /usr/local/lib/hy2/common.sh; hy2_ensure_runtime_dirs; hy2_acquire_lock_fd 7 "${HY2_LOCK_DIR}/temp.lock" 120 "节点锁繁忙"; export HY2_TEMP_LOCK_HELD=1; META="$(hy2_temp_meta_by_port "$PORT")"; [[ -n "$META" && -f "$META" ]] || hy2_die "未找到该端口的受管临时节点（不会删除主节点）"; TAG="$(hy2_meta_get "$META" TAG)"; [[ "$(hy2_meta_get "$META" PORT)" == "$PORT" && "$META" == "$(hy2_temp_meta_file "$TAG")" ]] || hy2_die "端口归属异常"; echo "正在删除端口 $PORT 的临时节点 $TAG"; FORCE=1 HY2_TEMP_LOCK_HELD=1 /usr/local/sbin/hy2_cleanup_one.sh "$TAG"; [[ ! -e "$META" ]] || hy2_die "删除未完成，请检查残留"; echo "端口 $PORT 的临时节点已删除"'
```

**安全边界：** 脚本使用与节点创建、删除相同的 `temp.lock`，并调用仓库已有的 `hy2_temp_meta_by_port` 校验唯一归属。如果端口不存在、只属于主节点、出现重复端口元数据或锁被占用，就拒绝删除；不会仅靠 `grep` 猜测 TAG。若端口的元数据已丢失，不能通过端口自动定位，请先只读排查并恢复受管状态，不能盲目删除服务文件。

成功后再次执行 `hy2_audit.sh` 和 `portbw report` 核查结果；如果清理遇到限速器等依赖故障，可能保留节点元数据以供修复，**不要把非零退出视为删除成功**。

### 清空全部临时节点

**下方命令会删除所有受管 HY2 临时节点，不应替代单端口删除。**

```bash
hy2_clear_all.sh
```

若部分节点清理失败，命令返回非零并保留未完成的元数据；主节点不受该命令影响。

</details>

<details>
<summary>独立 SOCKS5：按本机监听端口删除账号</summary>

仅在独立落地机执行：

```bash
socks5 del 41004
```

此命令不删除 HY2 节点；SOCKS5 的独立 `portbw` 策略如有配置，应另外核对是否需要撤销。

</details>

<a id="uninstall"></a>

<details>
<summary>卸载 HY2：先预览，按需保留或删除数据</summary>

在 HY2 VPS 执行。先下载并查看帮助、预览操作范围：

```bash
curl -fsSL --retry 3 https://raw.githubusercontent.com/liucong552-art/hy2jieshu/main/hy2-uninstall.sh -o /root/hy2-uninstall.sh
chmod 700 /root/hy2-uninstall.sh
bash -n /root/hy2-uninstall.sh && bash /root/hy2-uninstall.sh
```

默认只预览。确认备份已保存后，下面两种方式选择一种：

```bash
# 停止并移除 HY2 运行服务及大部分工具，保留数据、证书和 guard 恢复工具
bash /root/hy2-uninstall.sh --yes
```

```bash
# 同时删除 HY2 节点、配额、统计、配置及证书；并移除 Hysteria 核心
bash /root/hy2-uninstall.sh --yes --purge --remove-core
```

卸载会使 HY2 链接停止工作。WireGuard、NAT Peer、portbw 程序、acme.sh 账户与 SSH 配置分别管理，不随 HY2 卸载删除；保留数据卸载时也保留对应端口限速，`--purge` 会清理 HY2 主节点及临时节点元数据所列端口的 portbw 策略，其他端口保留。脚本对共用名称的配额服务和命令进行 HY2 归属检查，拒绝删除其他应用的组件。

保留数据卸载将临时 unit 备份到 root 私有的 `/var/lib/hy2/uninstalled-units/`，备份文件权限为 600；同时保留 `hy2-guard` 命令、停止/禁用的 guard unit 和 `/usr/local/sbin/hy2_nat_preflight.sh`，作为普通 / NAT 节点恢复工具，专用 nft 表会移除。重新安装管理器时，在 GC 之前恢复合法未到期 unit，先恢复 guard 规则与服务，确认防护后才启动保留节点。普通与 NAT 两节点的完整保留数据卸载/重装已在[本轮报告](tests/REAL_MACHINE_ACCEPTANCE_2026-11-15.md)通过外部客户端复测；前两次失败及修复过程仍保留。

`--purge` 删除的数据无法由 GitHub 恢复；`/root/.acme.sh` 中的账户及 ACME 证书副本仍保留，不能将此命令当作所有密钥的彻底销毁。保留数据卸载后重新安装也必须检查旧节点有效期和防护，不能将它当作空白机直接运行首次安装前置检查。

</details>

<a id="install"></a>

## 六、安装与更新

按组件展开。**WG-NAT 保留完整两端部署顺序，其他组件也保留完整安装和更新步骤。**

<details>
<summary>环境要求与各脚本运行位置</summary>

### 使用前准备

- HY2 / WG-NAT：Debian 11+ 或 Ubuntu 20.04+，使用 `root`，系统由 systemd 管理。
- 一个 A 记录指向**新 HY2 VPS** 的公网 IPv4；创建 IPv6 临时节点时，再准备相应 AAAA 记录。
- TCP `80` 可从公网访问以完成首次 TLS 证书签发；放行主节点 **UDP `443`** 和需要的临时节点 UDP `40000–50050`。
- WG-NAT 使用的 UDP 端口默认是 `51820`，需按两端实际防火墙与网络条件放行。
- Cloudflare 等 DNS 服务必须为 **DNS only**，不能让 HY2 的 UDP 连接走普通网页代理。
- 本文是 **Bash 命令**；域名、邮箱、公钥和 Peer 名称必须替换为真实值。

> **部署位置固定：** `hy2.sh`、`vpswg.sh`、`natjichang.sh` 在 **HY2 VPS** 上运行；`nat.sh` 只在 **NAT 出口机** 使用。已有正常运行的 NAT 出口机继续保留原配置和 Peer，不能为部署 HY2 重装管理器或覆盖 `/root/nat.sh`。各服务器按角色独立维护。

### 先确认运行位置

| 组件 / 文件 | 在哪台机器运行 | 用途 |
|---|---|---|
| `hy2.sh` | 新 HY2 VPS | 安装主节点、普通临时节点和管理组件 |
| `vpswg.sh` | 新 HY2 VPS | 配置 WG-NAT 的 VPS 端 |
| `natjichang.sh` | 新 HY2 VPS | 安装 WG-NAT 临时节点工具 |
| `nat.sh` | 独立 NAT 出口机 | 已部署则保留原管理器；仅在缺少 Peer 时按流程新增 |
| `traffic.py` | 新 HY2 VPS | 可选：UDP 每日流量统计 |
| `portbw-install.sh`、`portbw.py` | 需要限速的 VPS | 可选：安装、管理和恢复端口限速策略 |
| `hy2-guard.py`、`hy2-abuse.py` | 新 HY2 VPS | 可选：防火墙管理、异常审计 |
| `socks5-install.sh`、`socks5.py`、`socks5-traffic.py`、`socks5-audit.py` | 独立 SOCKS5 落地机 | 手动选择安装：账号、配额、流量及访问审计 |

`tests/` 和 `.github/workflows/` 是项目的测试与 CI，不需要复制到业务服务器。

**三台机器的职责，不要混淆：**

| 部署角色 | 实际安装的协议或工具 | 不应在这台机盲目执行 |
|---|---|---|
| HY2 入口 VPS | Hysteria2、HY2 节点/配额/IP 管理、可选 guard/abuse/portbw、必要时 VPS 端 WG | 不自动安装独立 SOCKS5、不初始化生产 NAT 出口 |
| WG/NAT 出口机 | 独立 `nat.sh`、WireGuard 接口和 Peer | 保留原有 NAT 管理器及 Peer；不为更新 HY2 重启共享 WG |
| SOCKS5 独立落地机 | 原样复用的 SOCKS5 安装器、账号、流量、访问审计；限速另行自选 portbw | 不运行 HY2 主安装器，SOCKS5 也不意味着 HY2 自动经 SOCKS5 转发 |

> README 中的安装、重装、删除、`watch`、`restore`、`probe-*`、`systemctl restart` 等命令均为**操作示例**，不是文档校验过程中应自动执行的步骤。共享生产系统先做快照并安排维护窗口。

需要单独查看系统准备和新 Peer 接入细节时，可参考[全新 VPS 部署手册](docs/FRESH_VPS_DEPLOY.md)；**下面 6.1、6.2 已经包含可直接执行的完整命令，不必跳转后才能安装。**

</details>

<a id="hy2-install"></a>

<details>
<summary>6.1 HY2 主节点：安装 → 配置 → 创建主节点 → 检查</summary>

### 1. 安装

在**全新 HY2 VPS** 执行：

```bash
apt-get update && apt-get install -y curl ca-certificates nano iproute2
```

依赖安装成功后，下载并安装管理工具（修改系统）：

```bash
curl -fsSL --retry 3 'https://raw.githubusercontent.com/liucong552-art/hy2jieshu/main/hy2.sh' -o /root/hy2.sh && chmod 700 /root/hy2.sh && bash -n /root/hy2.sh && bash /root/hy2.sh
```

本步骤生成 `/etc/default/hy2-main`、`/root/onekey_hy2_main_tls.sh`、临时节点管理器、配额和 IP 限制、看门狗、恢复任务以及对应 systemd 单元。**安装管理器本身还不会自动创建可连接的 HY2 主节点。**

### 2. 修改配置

在 **HY2 VPS** 执行：

```bash
nano /etc/default/hy2-main
```

下面是**配置文件内容示例**，写进编辑器，不是在终端执行的命令：

```ini
HY_DOMAIN=hy2.example.com
HY_IPV6_DOMAIN=
HY_LISTEN=0.0.0.0:443
ACME_EMAIL=admin@example.com
MASQ_URL=https://www.apple.com/
NODE_NAME=HY2-MAIN
TEMP_PORT_START=40000
TEMP_PORT_END=50050
ENABLE_SALAMANDER=0
HYSTERIA_VERSION=latest
HYSTERIA_UPDATE_POLICY=install-only
```

在 nano 中按 `Ctrl+O`、回车保存，再按 `Ctrl+X` 退出。

通常只需要修改：

- `HY_DOMAIN`：必须有 A 记录指向**当前新 VPS**，不能沿用 `helan` 的域名或旧连接参数。
- `ACME_EMAIL`：填写用于 ZeroSSL/ACME 注册及证书通知的有效邮箱；证书签发前必须确保 TCP/80 可达。
- `HY_IPV6_DOMAIN`：只有准备创建 IPv6 临时节点、且已经配置正确 AAAA 记录时才填写。
- `NODE_NAME`、`TEMP_PORT_START`、`TEMP_PORT_END`：按需调整。主节点默认使用 IPv4 UDP `443`；如果修改 `HY_LISTEN`，检查命令与防火墙也必须使用实际监听端口。

`install-only` 意味着后续重复执行主节点部署不会擅自把现有 Hysteria 核心升级到新版。此项目使用受信任的正式 TLS 证书，不以自签证书或客户端跳过证书验证作为正常安装步骤。

### 3. 创建或更新主节点

第一次签发正式 TLS 证书并创建 HY2 主节点，或修改上述配置后重新应用时执行：

```bash
bash /root/onekey_hy2_main_tls.sh
```

**重置 HY2 UDP/443 主节点认证密码（仅在确实需要轮换时执行）：** 默认重新部署主节点会保留已有密码；显式设置 `ROTATE_CREDENTIALS=1` 后，生成新的主节点认证密码、重新部署并更新链接，**旧主节点链接立即失效**。操作前请确认可以中断现有连接，并准备重新分发新链接；该操作不轮换临时节点密码、TLS 证书私钥或独立配置的 Salamander 混淆密码。

```bash
ROTATE_CREDENTIALS=1 bash /root/onekey_hy2_main_tls.sh
```

轮换成功后，新链接保存在 `/root/hy2_main_url.txt`，Base64 文件在 `/root/hy2_main_subscription_base64.txt`。这些文件都包含敏感凭据，只在可信环境中查看与分发；执行失败先检查日志和回滚状态，不要盲目重复轮换。**普通更新无需设置 `ROTATE_CREDENTIALS`。**

**这一步才会实际部署主节点。** 以后日常重新应用主节点，默认按配置的 `install-only` 策略复用已安装的 Hysteria 核心；切换域名之前应先完成新 DNS 和证书条件。

部署成功后查看主节点链接：

```bash
cat /root/hy2_main_url.txt
```

Base64 主节点订阅（需要时再查看）：

```bash
cat /root/hy2_main_subscription_base64.txt
```

链接包含密码，只在自己的客户端导入，不要发到 GitHub、公开日志或聊天截图。不同 VPS 应生成各自的主节点凭据，不复制其他机器的链接或密钥。

<details>
<summary>高级选择：显式升级 Hysteria 核心（日常不必执行）</summary>

升级前先建立 VPS 快照并确认正式维护窗口；不要用升级操作代替普通安装。将 `vX.Y.Z` 换成要升级的实际版本：

```bash
HYSTERIA_FORCE_UPDATE=1 HYSTERIA_VERSION=vX.Y.Z bash /root/onekey_hy2_main_tls.sh
```

确实需要官方当前稳定版时也可以指定 `HYSTERIA_VERSION=latest`。升级程序会预检候选核心、备份旧核心，在出错时尝试回滚；只有文件、核心和原活跃服务恢复验证成功才报告完成。回滚失败会返回非零、记录失败步骤并保留私有备份路径，需按日志处理；生产环境优先使用经过验收的确定版本和有效的 `HYSTERIA_BINARY_SHA256`。

**查看当前安装的 Hysteria 版本（只读，不会修改服务器）：**

```bash
hysteria version | grep '^Version:'
```

Hysteria 的版本子命令会先输出一个以空行开头的 Logo，再输出 `Version:` 字段；**不要使用 `head -n 1`**，它只会读取空行。上面的 `grep` 精确提取版本行，不安装、不更新、也不重启服务。

如果找不到该命令，先执行 `command -v hysteria` 检查安装状态，不要为了查询版本直接升级。

</details>

### 4. 怎样确认安装成功

在 **HY2 VPS** 执行：

```bash
systemctl is-active hy2.service
ss -lun '( sport = :443 )'
hy2_doctor.sh --strict
```

正常情况：`hy2.service` 显示 `active`，`ss` 能看到 **UDP/443** 监听，Doctor 没有关键错误。HY2 使用 UDP/QUIC，应检查 UDP 端口，不要用只查询 TCP 的 `ss -ltn` 判断节点是否在线。

然后将 `/root/hy2_main_url.txt` 中的完整 `hy2://` 链接导入 v2rayN 等兼容客户端，从**另一台设备**实际联网验证。**本机显示 active 和端口监听，不等于外部客户端一定可用。**

连接失败先查看[故障排查](#trouble-guide)，检查 A 记录、TCP/80 证书签发情况、UDP/443 防火墙和客户端配置，不要随意清空 nftables 或重新生成旧节点凭据。

</details>

<a id="wg-nat"></a>

<details>
<summary>6.2 WG-NAT：完整五步部署、NAT 管理与 Peer 探测</summary>

WG-NAT 让 HY2 VPS 上指定的临时节点通过独立 NAT 机器的公网 IPv4 出口访问互联网。**已有可用的 NAT 出口机无需重新初始化**；下面的步骤 1 仅适用于尚未部署 NAT 的新机器。每台新 HY2 VPS 必须拥有独立的 WireGuard 密钥与 Peer，不能复制其他服务器的私钥。

### 1. 初始化 NAT 出口机（只有全新、尚无 NAT 的机器才执行）

**已有 NAT 出口机已经通过 `/root/nat.sh` 验收时：跳过此步，不执行 `nat.sh init`，不覆盖原 `/root/nat.sh`。** 共享 NAT 上执行 `init/add/del` 可能重启整个 WireGuard 接口，影响其他 VPS。

如果未来另建一台**完全空白** NAT 出口机，才在那台 NAT 机执行下面的**新安装**示例：

```bash
apt-get update && apt-get install -y curl ca-certificates
curl -fsSL --retry 3 'https://raw.githubusercontent.com/liucong552-art/hy2jieshu/main/nat.sh' -o /root/nat.sh && chmod 700 /root/nat.sh && bash -n /root/nat.sh && bash /root/nat.sh init
```

无法自动识别 WAN 网卡时，仅在全新 NAT 出口机按实际名称指定 `WAN_IF=实际网卡`；不要用新装步骤修改已经稳定运行的 NAT 出口机。

### 2. 配置 HY2 VPS

在**新 HY2 VPS** 执行（下载并部署 VPS 端 WireGuard）：

```bash
curl -fsSL --retry 3 'https://raw.githubusercontent.com/liucong552-art/hy2jieshu/main/vpswg.sh' -o /root/vpswg.sh && chmod 700 /root/vpswg.sh && bash -n /root/vpswg.sh && bash /root/vpswg.sh
```

脚本会生成**新 VPS 专属**的 WireGuard 公钥。以后需要查询时，在该 HY2 VPS 执行：

```bash
cat /etc/wireguard/wg-nat.pub
```

### 3. 在已有 NAT 出口机添加新 VPS

先在**原 NAT 出口机**查已有 Peer：

```bash
bash /root/nat.sh list
```

如果新 HY2 VPS 已有**公钥、地址都正确**的独立 Peer，不要重复 `add`。只有确实缺少 Peer 时，才在共享 WireGuard 维护窗口内使用 **NAT 机现有的 `/root/nat.sh`** 添加（Peer 名不可与其他 VPS 重复）：

```bash
bash /root/nat.sh add hy2 hy2.example.com '这里替换成新HY2_VPS的WireGuard公钥'
```

三个参数分别是：

- `hy2`：这台新 VPS 在 NAT 机的**唯一 Peer 名称**，例如 `hy2-01` 或 `vps1`；切勿覆盖已有别的 VPS。
- `hy2.example.com`：**新 HY2 VPS** 的公网 IPv4 域名或地址。
- 第三个参数：在**新 HY2 VPS** 执行 `cat /etc/wireguard/wg-nat.pub` 得到的公钥，**不是** `helan` 的旧公钥。

NAT 机会分配 WG 地址并输出 VPS 的完整回填命令。**`nat.sh add` 可能重启共享 WG，导致其他 Peer 短暂断线。** 已有探测规则、timer 和防火墙继续由 NAT 机原有管理器维护，不再部署第二套 NAT 管理器。

### 4. 回到 HY2 VPS 执行回填命令

在**新 HY2 VPS** 原样执行 NAT 机刚刚打印的 `wg_nat_set_peer.sh` 回填命令，**不照抄测试机的公钥、隧道地址或示例凭据**。如忘记 NAT 公钥，可只在 NAT 出口机查询：

```bash
cat /etc/wireguard/wg-exit.pub
```

新 HY2 VPS 回填后检查 WG 出口：

```bash
/usr/local/sbin/wg_nat_healthcheck.sh
```

**注意：这条健康检查是主动修复型检查**：它可能调整本机 rp_filter、并调用 WG guard 修复策略路由/防火墙；**不是只读操作**。仅需不修改系统的状态核验，可运行 `bash /root/hy2_fresh_preflight.sh wg`（需先按[全新 VPS 手册](docs/FRESH_VPS_DEPLOY.md)下载），它只查询最近握手和 mark 路由，不会实际发起 NAT 出口 IP 探测。

成功时应出现类似 `OK EXIT_IP=...`。若失败，先确认新 Peer 公钥、AllowedIPs、UDP/51820 和握手情况，不要在已有 NAT 机上重复执行 `init`。

### 5. 安装 WG-NAT 临时节点工具

在**新 HY2 VPS** 执行（只更新 VPS 端模块，不动 NAT 机）：

```bash
curl -fsSL --retry 3 'https://raw.githubusercontent.com/liucong552-art/hy2jieshu/main/natjichang.sh' -o /root/natjichang.sh && chmod 700 /root/natjichang.sh && bash -n /root/natjichang.sh && bash /root/natjichang.sh
```

检查新 VPS 的 WG-NAT 准备情况：

```bash
bash /root/natjichang.sh check
```

配置完成后，需要 NAT 出口的 HY2 临时节点使用 `hy2_mktemp_nat.sh`，普通 HY2 临时节点仍使用 `hy2_mktemp.sh`。

### NAT 出口机日常管理

以下命令**仅在现有 NAT 出口机**执行，使用该机器已经安装的 `/root/nat.sh`：

#### 查询已有 Peer 与运行状态

```bash
bash /root/nat.sh list
bash /root/nat.sh status
```

#### 更新或移除某台 VPS

只有确实需要变更 Peer 时才执行 `add` 或 `del`；它们可能重启整个共享 WG。先用 `list` 找到真实 Peer 名，不要把例子中的 `hy2` 当作现有节点：

```bash
# 更新已存在的同名 Peer（仅在维护窗口）
bash /root/nat.sh add hy2 hy2.example.com '新的VPS的WireGuard公钥'
# 删除某个指定 Peer（危险：此 Peer 会断线）
bash /root/nat.sh del hy2
```

### 单 Peer 探测（可选）

已有 NAT 出口机如果配置了 Peer 探测，就使用当前管理器查询状态，**不要因部署 HY2 重装探测器或调整正在正常运行的 timer**。相关状态需要在目标 NAT 机器上核对：

```bash
bash /root/nat.sh probe-once hy2
bash /root/nat.sh probe-status hy2
```

确实需要控制已有 Peer 的定时器时，再使用 `probe-enable` / `probe-disable`；**不要为了 HY2 验收去启停已经工作的探测定时器**。`WG_PROBE_SENT` 仅表示探测包已尝试发送，不是远端已成功重新握手的证明。

**可选探测的操作边界（仅供确认需要时使用）：**

| 命令 | 作用 | 生产共享 NAT 注意事项 |
|---|---|---|
| `bash /root/nat.sh probe-status hy2` | 查询指定 Peer 探测状态 | 优先使用，避免改变已有 timer |
| `bash /root/nat.sh probe-once hy2` | 主动尝试发送一次探测 | 不是只读；是否成功应再看握手和真实出口 |
| `bash /root/nat.sh probe-enable hy2` | 为指定 Peer 启用探测 | 可能更改定时器；仅在明确授权和维护窗口执行 |
| `bash /root/nat.sh probe-disable hy2` | 为指定 Peer 禁用探测 | 会改变监控行为；没有明确维护需求不应执行 |

运行 `list` / `status` 前先确认登录的是**落地 NAT 出口机**，不是新 HY2 VPS。共享出口机已有真实用户时，不执行 `init`、无目的的 `add` / `del`、WG 重启或“为了演示命令”的探测操作；新 VPS 的独立 Peer 已存在就直接复用。

</details>

<a id="daily-install"></a>

<details>
<summary>6.3 HY2 每日流量：独立安装 / 更新</summary>

在已经安装 HY2 管理器的 **HY2 VPS** 上，以 `root` 执行。独立统计组件支持主节点、普通临时节点及 WG-NAT 临时节点；按 UDP 端口统计，无需重新安装主节点或重启 Hysteria。

### 安装 / 更新

在 HY2 VPS 下载并安装统计组件（首次安装和后续更新使用同一命令）：

```bash
curl -fsSL --retry 3 'https://raw.githubusercontent.com/liucong552-art/hy2jieshu/main/traffic.py' -o /root/hy2_traffic.py && python3 -m py_compile /root/hy2_traffic.py && python3 /root/hy2_traffic.py --install
```

安装器会管理 `hy2-traffic.timer`、`hy2-traffic.service` 及 `hy2-traffic-shutdown.service`。旧的历史数据应当保留；不需要执行 `hy2.sh` 重装，也不会更改 `pq-save.timer`。

### 检查统计与历史

```bash
systemctl is-active hy2-traffic.timer hy2-traffic-shutdown.service
systemctl list-timers --all hy2-traffic.timer
hy2_traffic
hy2_traffic main
hy2_traffic 40020 --json
```

`40020` 是示例端口，请先从 `hy2_audit.sh` 查到真实端口。`hy2-traffic.service` 为定时 oneshot，执行结束后显示 inactive 属正常情况；关机保存服务使用 `RemainAfterExit=yes`。具体日流量与配额可能采样时间不同，先核对采样时间，不要直接当成实时限速数字。

</details>

<a id="portbw-install"></a>

<details>
<summary>6.4 portbw：完整首次安装、受管理版本升级与旧版本迁移</summary>

### 安装 / 更新 portbw（需要同时下载两个文件）

**运行位置：需要限速的服务器。首次安装和升级分开操作。**

1. 仅限没有旧 portbw 的新安装环境，先安装依赖（修改系统）：

```bash
apt-get update && apt-get install -y curl ca-certificates python3 iproute2 nftables util-linux coreutils procps
```

依赖安装成功后，下载两个配套文件；以下命令仅下载，不安装、不创建限速策略：

```bash
curl -fsSL 'https://raw.githubusercontent.com/liucong552-art/hy2jieshu/main/portbw-install.sh' -o /root/portbw-install.sh && \
curl -fsSL 'https://raw.githubusercontent.com/liucong552-art/hy2jieshu/main/portbw.py' -o /root/portbw.py
```

两次下载均成功后再继续。已有 portbw 的主机应看下面的升级说明，不要执行新安装流程。

2. 预检环境和配套源码（不安装 portbw）：

```bash
bash /root/portbw-install.sh check
```

**只在出现 `CHECK PASS` 后继续。** 若提示已有安装，不要删旧文件来强行通过，改看下面的升级 / 迁移说明。

3. 执行实际安装（修改系统）：

```bash
bash /root/portbw-install.sh install
```

应出现 `INSTALL PASS` 和 `VERIFY PASS`；仅下载文件、语法检查或运行 `check` 都不会完成安装。首次安装不会自动创建端口限速策略。

4. 核对安装结果：

```bash
command -v portbw
bash /root/portbw-install.sh verify
systemctl is-active portbw-watch.service
portbw report
portbw audit
```

命令路径应为 `/usr/local/sbin/portbw`，Watchdog 通常为 `active`；尚未设置端口时列表为空属于正常情况。核对通过后，选择后面的静态设置或自动设置。

如预检要求指定物理网卡，先用 `ip -4 route get 1.1.1.1` 查看，再在 `check` 和 `install` 后分别加 `--iface eth0`，把 `eth0` 换成实际物理网卡名；不要根据示例猜网卡。

<details>
<summary>已有受管理版本：升级方法</summary>

**升级已由本系列通用安装器管理的已知 v1/v2/候选版：** 先按上方两条下载命令重新下载同一版安装器和源码，先记录当前的 `portbw report` 与 `portbw audit`，保存可恢复的 VPS 快照，再在维护窗口执行（不要重新执行首次安装 `install`）：

下面三步用 `&&` 串联，前一步失败时不会继续下一步；预检失败时先处理原因，不能跳过：

```bash
bash /root/portbw-install.sh preflight-upgrade && \
bash /root/portbw-install.sh upgrade && \
bash /root/portbw-install.sh verify
```

全部成功后核对：

```bash
portbw report
portbw audit
```

首次安装的 `check` 要求干净环境，已有受管理安装要用 **`preflight-upgrade`**。升级工具只接受有 `/var/lib/portbw-resident-release/current.env` 管理记录，且已安装源码 SHA 与记录一致的受支持旧版；未知来源的旧版即使已有端口配置也**不允许直接覆盖或伪造管理记录**。安装器会持久保存备份，但不能取代 VPS 快照。不要执行 `nft flush ruleset`、`tc qdisc del ... root`，不要停止其他业务的 nft/tc 系统规则。

</details>

<details>
<summary>其他旧仓库 / 无管理记录版本：迁移说明</summary>

**其他旧仓库/无管理记录版本的迁移（不能直接 `install`/`upgrade`）：** 特别是 `zuizhongheji` 等独立旧版及使用旧 tc flower 槽位的机器，先记录每个端口的**基础上/下行、降速上/下行、触发百分比、持续秒数、滚动保护、冷却**，执行 `portbw report` 与 `portbw audit` 并制作 VPS 快照。只可在维护窗口按照确切旧版校验过的迁移工具或逐端口安全删除方法移除旧限速，再安装此通用版并重新添加规则；在删除到重建期间可能暂时失去限速保护。不要把普通安装器用于未知来源旧版本，更不要手动清空整个 nftables ruleset、网卡 root qdisc 或其他业务规则。

</details>

</details>

<a id="socks5"></a>

<details>

<summary>6.5 SOCKS5：独立可选落地、首次安装、更新与每日流量</summary>

本仓库提供一套可选的 SOCKS5 管理工具，**仅用于独立落地机手动安装**。`hy2.sh`、`nat.sh`、`vpswg.sh`、`natjichang.sh` 均不会自动安装 SOCKS5，也不会让 HY2 流量自动经过 SOCKS5。此模块只支持 TCP CONNECT，不支持 UDP ASSOCIATE。

**验收范围：** SOCKS5 工具的历史验证和源码校验见[组件复用与完整性记录](docs/SOCKS5_UPSTREAM_REUSE.md)。独立 SOCKS5 落地机未在本轮 HY2 VPS 上重新部署或进行完整功能测试，不能因此声称所有 Debian/Ubuntu 版本和升级场景均已通过。

### 1. 四文件同目录下载

以下在**独立 SOCKS5 落地机**以 root 执行；需要安装时才手动运行。先准备下载工具：

```bash
apt-get update && apt-get install -y curl ca-certificates
```

四个文件保持配套，下载失败不继续：

```bash
install -d -m 700 /root/socks5-release && \
curl -fsSL --retry 3 https://raw.githubusercontent.com/liucong552-art/hy2jieshu/main/socks5-install.sh -o /root/socks5-release/socks5-install.sh && \
curl -fsSL --retry 3 https://raw.githubusercontent.com/liucong552-art/hy2jieshu/main/socks5.py -o /root/socks5-release/socks5.py && \
curl -fsSL --retry 3 https://raw.githubusercontent.com/liucong552-art/hy2jieshu/main/socks5-traffic.py -o /root/socks5-release/socks5-traffic.py && \
curl -fsSL --retry 3 https://raw.githubusercontent.com/liucong552-art/hy2jieshu/main/socks5-audit.py -o /root/socks5-release/socks5-audit.py
```

### 2. 首次安装 / 已安装版本更新

首次安装：

```bash
bash -n /root/socks5-release/socks5-install.sh && AUTO_DEPS=1 bash /root/socks5-release/socks5-install.sh install
```

安装器优先使用自身目录内的三个配套模块，并在安装前校验 SHA256；必须看到 `[socks5] SHA256 校验通过`。不匹配应立即停止检查文件版本，不可跳过校验。**安装器仍保留独立的预设远程备用来源，并不保证回退至本仓库**；因此必须按上面的步骤下载齐全四个同版本文件，以优先使用本仓库配套的本地模块。

家宽路由器后的主机可在 `install` 后加 `--host proxy.example.com` 指定真实公网入口；必要时加 `--wan-if 实际网卡`。账号的公网/本机端口映射按[新建 SOCKS5 账号](#socks5-create)设置。

已有账号的主机先保存可恢复快照，按上面下载流程更新四文件，再执行 `update`；不要对已有安装套用首次安装命令：

```bash
bash -n /root/socks5-release/socks5-install.sh && AUTO_DEPS=1 bash /root/socks5-release/socks5-install.sh update
```

更新不会主动清空账号，但迁移访问日志配置可能短暂重启实例；更新后核对账号、配额和实际连接。原安装器没有通用卸载/降级子命令，失败回退使用已验证的整机快照，不编造 `socks5 uninstall`。

**已有生产落地机安全升级核对顺序：** 先保存服务商快照及原账号列表、用量与本机端口，再核对四文件的 SHA/本地校验状态；只有在确实需要升级、获得授权并预留维护窗口时才执行 `update`。更新期间可能短暂重启实例，事后需核对原账号、配额、IP 占位、日志及真实连接。若保持原样且没有升级需求，**什么都不运行**。

SOCKS5 的访问审计、`socks5-traffic --install` 与独立 `portbw` 均在落地机分别管理，不能把 HY2 VPS 上的 UDP/443 或临时节点端口作为 SOCKS5 本机 TCP 端口使用；两种协议在同一个仓库出现，不代表它们共享用户数据库或自动转发链路。

### 3. 每日流量与管理入口

```bash
socks5-traffic --install
socks5 list
socks5 status
```

每日统计独立开启；[账号/日志查询](#socks5-query)、[创建](#socks5-create)、配额和限速分别使用本 README 对应命令。运行数据位于 `/etc/socks5-manager/` 与 `/var/lib/socks5-manager/`，包含凭据，备份及审计原始日志不要公开。

</details>

<a id="maintenance"></a>

<details>
<summary>6.6 HY2 / WG-NAT / NAT / 每日统计的更新方法</summary>

> **更新范围必须区分：** `hy2.sh` 只更新 HY2 管理组件，不自动更新 Hysteria 核心、VPS WireGuard、NAT 机 `nat.sh` 或通用 `portbw`。GitHub `main` 变化不会自动改动已安装 VPS。生产环境优先使用确认过的提交版本，升级前保存可恢复的 VPS 快照。

#### HY2 VPS：更新管理工具

在**需要更新的 HY2 VPS** 下载同一仓库版本，并检查 Bash 语法后再更新（保留既有主节点配置、节点状态和凭据；仍建议先做 VPS 快照）：

```bash
curl -fsSL --retry 3 \
  'https://raw.githubusercontent.com/liucong552-art/hy2jieshu/main/hy2.sh' \
  -o /root/hy2.sh && \
chmod 700 /root/hy2.sh && \
bash -n /root/hy2.sh && bash /root/hy2.sh
```

需要重新应用主配置或**显式**升级 Hysteria 二进制时，回到上面的[6.1 主节点步骤](#hy2-install)。仅刷新管理组件不应当自动切换正在运行的核心版本。

#### HY2 VPS：更新每日流量统计

```bash
curl -fsSL --retry 3 \
  'https://raw.githubusercontent.com/liucong552-art/hy2jieshu/main/traffic.py' \
  -o /root/hy2_traffic.py && \
python3 -m py_compile /root/hy2_traffic.py && \
python3 /root/hy2_traffic.py --install
hy2_traffic
```

只更新统计模块，无需删除历史数据或重建节点。

#### HY2 VPS：更新 WireGuard / NAT 临时节点工具

这些脚本**只在 HY2 VPS** 执行，不是 NAT 出口机的更新。WG 脚本可能调整路由与接口配置；服务稳定时不应盲目覆盖，确需升级应安排维护窗口：

```bash
curl -fsSL --retry 3 \
  'https://raw.githubusercontent.com/liucong552-art/hy2jieshu/main/vpswg.sh' \
  -o /root/vpswg.sh && \
chmod 700 /root/vpswg.sh && \
bash -n /root/vpswg.sh && bash /root/vpswg.sh

curl -fsSL --retry 3 \
  'https://raw.githubusercontent.com/liucong552-art/hy2jieshu/main/natjichang.sh' \
  -o /root/natjichang.sh && \
chmod 700 /root/natjichang.sh && \
bash -n /root/natjichang.sh && bash /root/natjichang.sh
```

更新后用 `wg_nat_healthcheck.sh` 与 `natjichang.sh check` 核实；现有 Peer 公钥、AllowedIPs 与 NAT 映射必须保持正确。

#### NAT 出口机：保留现有部署，不直接覆盖

**现有 NAT 出口机不需要为了接入 HY2 而升级或重装。** 必须区分“查看已有状态”“备份”“更新管理文件”与“重新初始化 WG”：检查和备份不应导致 WG 重启；后两项必须由 NAT 管理员在维护窗口实施。**不要在生产 NAT 上重复执行 `nat.sh init`，也不要安装第二套探测 timer。**

<details>
<summary>NAT 出口机：只读检查现有脚本、接口与 Peer（不执行 init）</summary>

```bash
bash -n /root/nat.sh
sha256sum /root/nat.sh
bash /root/nat.sh list
bash /root/nat.sh status
wg show wg-exit
```

`sha256sum` 只显示当前已安装脚本的哈希，并不自动判断是否需要升级。若要与候选新版本比较，必须先取得来源和版本明确的候选文件，再计算哈希、审查变更并做好备份。哈希不同**不是**立即覆盖生产 NAT 的理由。

</details>

<details>
<summary>NAT 管理员维护前备份 WG 与防火墙状态</summary>

```bash
umask 077
mkdir -p /root/wg-backups
BACKUP_TS=$(date +%Y%m%d-%H%M%S)
tar -C /etc -czf "/root/wg-backups/wireguard-${BACKUP_TS}.tar.gz" wireguard
iptables-save > "/root/wg-backups/iptables-${BACKUP_TS}.rules"
```

备份包含 WireGuard 私钥，不要上传公开仓库。`nat.sh init/add/del` 可能重启共享 WG 接口并影响全部 Peer，不能把它们当作只更新脚本文件的命令。生产环境先记录现有 Peer 的公钥、AllowedIPs 与映射关系，安排维护窗口并准备恢复方案。

</details>

#### 通用 portbw：单独的受管理更新流程

`portbw.py` 与 `portbw-install.sh` 是本仓库提供的通用限速组件。已有受管理版本应按 [6.4 portbw 安装与升级](#portbw-install) 的 `preflight-upgrade → upgrade → verify` 顺序维护，**不要在已安装的环境重新执行首次安装 `install`**。升级前核对当前策略与管理记录，并完成备份。

</details>

**各组件的更新范围：** HY2 管理器、每日流量、portbw、WG/NAT、独立 SOCKS5 分别有自己的更新入口和状态目录。仅更新 `hy2.sh` 不意味着应修改共享生产 NAT Peer 或强制升级 SOCKS5；单独更新脚本前先核对版本和已保存状态。新部署与已有业务机器必须区分：`install` 可能初始化服务，`update`/`upgrade` 可能重启已有业务，`check`/`audit` 通常负责检查但也应先核实实际脚本接口。

<a id="abuse-install"></a>

<details>
<summary>6.7 HY2 访问与异常审计：独立安装和实时 Stats API（可选）</summary>

在 HY2 VPS 以 root 安装只读命令；**不会重启 Hysteria、修改服务配置或启动后台采集器**：

```bash
curl -fL --retry 3 https://raw.githubusercontent.com/liucong552-art/hy2jieshu/main/hy2-abuse.py -o /root/hy2-abuse.py
python3 -m py_compile /root/hy2-abuse.py
install -m 755 /root/hy2-abuse.py /usr/local/sbin/hy2-abuse
PORT=40020 DAYS=7 hy2-abuse
hy2-abuse 40020 --days 7 --json
```

可选：**只有你已在 Hysteria 配置里自行启用 `trafficStats`、并仅监听本机回环地址**时，才能额外读取当前 TCP 连接目标：

```bash
# secret 文件路径为示例，务必 600 权限且由 root 保管
hy2-abuse 40020 --days 7 \
  --stats-url http://127.0.0.1:9999 \
  --secret-file /root/hy2-stats-secret
```

本程序不会代你创建 API 服务或暴露公网端口，也不保存 API 密码。只允许字面量 `127.0.0.1` 或 `[::1]`，拒绝远程地址、HTTP 跳转和代理转发。使用 `--secret-file` 前，需确保该文件与配置中的 API secret 一致且为 root 私有。

`/online` 和 `/dump/streams` 返回的是**当前**在线与 TCP 流，不是 7 天的访问域名历史；`DAYS` 只作用于 journald 异常事件。不要把这个工具的汇总当作自动处罚依据。

</details>

<a id="backup-recovery"></a>

<details>

<summary>备份位置与回退方法</summary>

操作前建立可恢复的整机快照。GitHub ZIP 不包含运行中的节点、内核规则和流量状态。

| 组件 | 默认位置 | 说明 |
|---|---|---|
| HY2 主节点及证书 | `/etc/default/hy2-main`、`/etc/hysteria/` | 包含密钥 |
| HY2 配额/IP/节点状态 | `/var/lib/hy2/` | 包含节点密码 |
| WireGuard | `/etc/wireguard/` | 两端分别备份，包含私钥 |
| portbw | `/etc/portbw/`、`/var/lib/portbw/`、`/var/lib/portbw-resident-release/` | 独立的限速配置和备份 |
| SOCKS5 独立落地机 | `/etc/socks5-manager/`、`/var/lib/socks5-manager/` | 只在可选 SOCKS5 落地机备份，含账号密码与使用状态 |
| 日志与验收证据 | `journalctl`、本机 root 私有验收目录 | journald 可轮替，日志范围不等于永久完整历史 |

<details>

<summary>受管理 portbw 专用回滚</summary>

```bash

bash /root/portbw-install.sh rollback

```

仅适用于当前安装器认可的托管记录和有效备份；回滚后仍需执行 `portbw audit` 和客户端连接测试。

</details>

</details>

**恢复与更新的安全边界：**

1. 更新前先创建服务商控制台能够实际恢复的整机快照；GitHub 源码备份不等于服务器业务状态备份。
2. HY2 主节点、受管理临时节点与证书、配额/IP 元数据分别备份，确认私钥和账号资料只保存在 root 私有目录，不要提交到 GitHub。
3. 通用 `portbw` 的 `rollback` **只针对安装器识别的受管理版本**，与 HY2 核心回滚、SOCKS5 更新回退无关；不要错误地用一个组件的恢复命令处理另一个组件。
4. SOCKS5 原安装器不提供通用 `uninstall` / `rollback`；涉及生产账号和日志变更，应使用预先验证的整机快照恢复，不能用旧脚本盲目覆盖。
5. 核心或组件恢复后，应先核对规则、服务及真实客户端；保护规则缺失时应维持 fail-closed，不应为恢复连接而临时开放全部来源。
6. 已完成的 Debian 13 真机测试与历史可复用证据见[本轮验收](#acceptance)，未测的真实 CA 续期、IPv6、72 小时/30 天长周期及异常断电仍应明确标记，不因 README 改写而变为“已通过”。

<a id="release-notes"></a>

<details>

<summary>版本说明与看门狗运行规则</summary>

历史测试机在 2026-11-04 Debian 13 环境中验证了 HY2 主/临时节点、双向配额、来源 IP 限制、到期清理、WG-NAT 出口、断线防泄漏、VPS 重启恢复和 `portbw` 静态 10/10 Mbps。自动触发与恢复的历史验证范围见[组件审计](tests/SHARED_COMPONENT_AUDIT_2026-11-05.md)；旧记录不代表当前 VPS 已完成持续高负载测试。

首次安装后配额重启归零问题由 [首次关机保存修复](https://github.com/liucong552-art/hy2jieshu/commit/893bb2dc25cc96bd7076849385d79ebefd730952) 修复；通用限速组件的引入与适配记录见[项目提交](https://github.com/liucong552-art/hy2jieshu/commit/ab009ef736dc77c3cf6cf00f724867902ae89c04)。

### 看门狗与生产运行规则

- **HY2 临时节点 Watchdog 72 小时宽限：** 第三次连续恢复失败后，才开始一次持久化的 72 小时宽限期。系统重启/手工重启服务不重置 `WATCHDOG_GRACE_START_EPOCH`；只有服务、端口监听、全部配额/IP 防护，且 NAT 节点的 WG 路由/握手均被重新核实为健康，才清空失败次数和宽限计时。
- **宽限期间仍然 fail-closed：** 若缺失 nft 防护，停掉监听并保留节点；若 WG 无有效握手，恢复失败的 NAT 节点不允许继续监听，不能改走 VPS 公网。
- **到期处理：** 自然 TTL 到期按原规则正常删除；72 小时宽限结束后，只有再次恢复失败且本次启动已稳定至少 300 秒，才执行最终节点清理。**仍需在专用临时节点上进行跨重启、完整 72 小时或模拟时钟验收**，GitHub CI 不能代替真实故障链路验证。
- `portbw` 是共享组件；HY2 节点删除/到期只清理对应端口策略，失败保留元数据供重试，其他端口不受影响。
- `nat.sh init/add/del` 可能重启 NAT 机 WG，影响其他 Peer。

</details>

<a id="acceptance"></a>
### 验收记录：当前新机与历史证据分开

2026-11-15 Debian 13 新 VPS 的正式 TLS 主节点、临时节点、普通/WG-NAT 两种出口、UDP 业务、真实配额耗尽与恢复、来源 IP 限制、暂停/黑名单、重启保留、卸载重装及错误回滚等实测，见[本轮完整报告](tests/REAL_MACHINE_ACCEPTANCE_2026-11-15.md)与 [TASK_PROGRESS.md](TASK_PROGRESS.md)。2026-11-04/05 旧测试机结果见[历史验收](tests/REAL_MACHINE_ACCEPTANCE_2026-11-04.md)，**不自动计入本轮通过**。

| 状态 | 判定原则 |
|---|---|
| 当前真机通过 | 当次有外部客户端、服务及内核状态的可核查记录 |
| 复用已有证据 | 仅限同一组件源码、依赖和使用条件一致的场景 |
| 仅自动化测试通过 | 脚本/模拟场景通过，不等于真实服务器或客户端通过 |
| 尚未验证 | IPv6、实际 72 小时/30 天周期、CA 真实续证、异常断电等保留边界 |
| 不适用 | Reality UUID/密钥与 Xray 专属规则；SOCKS5 是另行安装的 TCP CONNECT 落地组件 |

本期剩余额度一行命令目前只有隔离模拟验证，**没有真实 VPS 验收**；不能把 GitHub CI 成功当作真机业务确认。

<a id="feature-gap"></a>
### 组件能力与使用边界

| 组件 | 本项目提供的能力 | 使用前需要注意 |
|---|---|---|
| HY2 主节点 / TLS | Hysteria 2、正式 TLS、UDP/443、主节点密码轮换 | 轮换后旧链接失效；证书、SNI 和公网 UDP 放行需核对 |
| 临时节点 / 配额 | 独立密码、到期时间、IP 槽位、UDP 双向总配额及每日统计 | 当日流量历史与总配额不同；本期余额调整尚未真机验收 |
| WG-NAT | 独立 Peer、WireGuard 路由、握手检查和出口防泄漏 | 生产 NAT 不因 HY2 更新而重新初始化；配置前须确认 Peer |
| 防火墙与限速 | `hy2-guard` 端口暂停、IP/CIDR 黑名单及 `portbw` 带宽控制 | 规则覆盖指定数字端口；避免误用 SSH 管理端口 |
| 独立 SOCKS5 | 可选的单独落地机账号、配额、流量和访问审计 | 需单独手动安装，仅 TCP CONNECT，不支持 UDP ASSOCIATE |
| 访问与异常审计 | 来源 IP 占位、journal 异常分类、可选实时 TCP 目标 | 不提供完整历史访问域名、UDP 历史目标或自动处罚 |

功能验证与未覆盖项分别见[本轮真机报告](tests/REAL_MACHINE_ACCEPTANCE_2026-11-15.md)、[组件复用审计](tests/SHARED_COMPONENT_AUDIT_2026-11-05.md)和[历史验收记录](docs/LEGACY_ACCEPTANCE_GUIDE.md)。

仅在自己拥有或已获授权的服务器使用。NAT 机原有 WireGuard 配置、Peer 以及独立 SOCKS5 的用户状态都不会因 README 变化而自动更新。
