# otssh — 一条命令，发一张限时 SSH 通行证

[English](README.en.md) | **中文**

![Go](https://img.shields.io/badge/Go-1.22%2B-00ADD8?logo=go&logoColor=white)
![Platform](https://img.shields.io/badge/platform-linux-lightgrey)
![License](https://img.shields.io/badge/license-MIT-blue)
![Binary](https://img.shields.io/badge/binary-4MB%20static-brightgreen)
![Tests](https://img.shields.io/badge/tests-16%20unit%20%2B%2030%20e2e-success)

```
$ otssh grant --ttl 10m

凭证 id      f6yp83wh
有效期       10m0s   到期 2026-10-08 09:30:00 CST
权限         有效期内可重复使用 · 禁止端口转发
服务端       203.0.113.10:2222

  ssh ot1a1b2c3d4e5f6g7h8i9j0k...@203.0.113.10 -p 2222

把上面那行整条发给对方即可；到点服务端会主动断开连接。
```
![Uploading image.png…]()

![Uploading image.png…]()

对方拿到这一行就能登录，**不需要密码、不需要公钥、不需要改任何 sshd 配置**。
时间一到，哪怕他正在 `vim` 里敲字、正在跑 `sleep 100000`，**服务端直接把会话连根掐掉**。

---

## 为什么需要它

把服务器交给别人时——外包、临时运维、AI Agent、从中转商/二手市场租来的机器——
你实际交出去的是**长期凭证**：root 密码、私钥，或者一条永久有效的 `authorized_keys`。

麻烦不在于"他能不能进来"，而在于：

- 他留了一份档，三个月后你还不知道；
- 他把账号转手卖了，买家随时能进；
- 你事后想收回，得改密码、删公钥、重启 sshd，还可能漏掉几个地方；
- 他进来干了什么，你没有任何记录。

otssh 换个做法：**不给凭证，给一张限时通行证。**

```bash
# 服务端装的时候就把来源卡死（可选，但公网强烈建议）
sudo bash install.sh --allow-from 203.0.113.7/32

# 之后每次要放人进来
sudo otssh grant --ttl 10m --once
# → ssh ot1a1b2c3d4e5...@your-server -p 2222
```

这一行就是全部。对方拿到就能登录，**10 分钟后自动失效，而且只能用一次**。
他留档没用，转卖没用——过了这个点，那串字符就是一堆废字符。

### 能防什么、防不了什么

说清楚边界，免得你对它期待过高：

| | |
|---|---|
| ✅ 凭证留档 / 转卖 / 复用 | 有效期只有几分钟，`--once` 用过即废 |
| ✅ 权限过大 | `--run-as` 可以给非 root 用户，而不是 root |
| ✅ 事后收不回 | `otssh init --force` 轮换密钥，所有已发凭证立即失效——不用改密码、不用碰 sshd |
| ✅ "干了什么不知道" | JSONL 审计（谁、何时、从哪来、待了多久、跑了什么命令）+ 可选会话录制 |
| ✅ 中途被第三方截获 | 短 TTL + `--allow-from` 绑来源 IP，窗口只有几分钟 |
| ❌ **宿主机的主人** | 如果机器本身就是他开的，他随时能读 `/etc/otssh/secret`、或直接装个后门。这只能在机器**之外**解决：自己控宿主、别在上面放敏感数据、全盘加密 |
| ❌ 同一宿主的流量嗅探 | TTL 只是把可利用窗口压到几分钟，不是消除 |

一句话：**otssh 把你交出去的访问权从"永久"压缩到"几分钟"，并且留下痕迹。
它没法把一个本来就被人攥着 root 的宿主机变成安全的机器。**

---

## 目录

- [为什么需要它](#为什么需要它)
- [30 秒上手](#30-秒上手)
- [它是怎么做到的](#它是怎么做到的)
- [命令参考](#命令参考)
- [部署到 Linux](#部署到-linux)
- [安全模型](#安全模型) ← **用之前务必看这节**
- [和现成方案比](#和现成方案比)
- [测试](#测试)
- [已知限制](#已知限制)

---

## 30 秒上手

**服务器上一条命令装完**（包里带预编译二进制，目标机不需要 Go、不需要联网）：

```bash
unzip -q otssh-0.1.0-linux.zip && sudo bash otssh-0.1.0/install.sh
```

装完就可以把 zip 和解压目录一起删掉，安装不受影响。接着签发第一张凭证：

```bash
sudo otssh grant --ttl 10m     # 要读 /etc/otssh/secret，所以需要 root
```

<details>
<summary>想先在本机试跑、不装成服务</summary>

```bash
make build          # 本机装了 Go 1.22+ 就行，产物是静态单文件
sudo ./otssh init --dir /etc/otssh
sudo ./otssh serve --listen :2222 --run-as root     # 前台跑，方便看日志
otssh grant --ttl 10m                               # 另开一个终端
```
</details>

把 `grant` 打印出来的那行 `ssh ot1...@host -p 2222` 整条发出去，对方复制粘贴回车即可。
第一次连接会问一次主机指纹，答 `yes` 就行（建议先用带外方式把指纹给他，见[安全模型](#安全模型)）。

---

## 它是怎么做到的

### 核心：凭证是**自包含**的，服务端**无状态**

```
ot1 <exp:8> <flags:2> <nonce:8> <mac:16>          共 37 字符，全小写 base36
└┬┘ └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘
 │     │       │       │       └── HMAC-SHA256(secret, 前 21 字符) 取前 10 字节
 │     │       │       └────────── 随机数：审计标识 + "一次性"作废的依据
 │     │       └────────────────── 权限位（一次性 / 允不允许 -L / 录不录制）
 │     └────────────────────────── 绝对到期时间（Unix 秒）
 └──────────────────────────────── 版本前缀

     整串直接当 SSH 用户名用：  ssh <这串>@host -p 2222
```

服务端只需要持有同一个 `secret`：比对 HMAC，看时间没到，就放行。
**不查库、不联网、不需要任何中心服务、服务重启也不丢凭证**。

唯一需要状态的是 `--once`（用过即废），它记在本地 `used.log` 里。

### 为什么非得把凭证塞进用户名

SSH 协议**不发送目标主机名**（不像 HTTPS 有 SNI）。客户端连过来，服务端只能看到
TCP 五元组 + 认证请求里的 `username`。所以要做"一个端口接管所有凭证"，
用户名就是唯一能用的字段。

这也是 `ssh git@github.com` 里那个 `git` 的来源——同一个套路，只不过那里是固定标签。

### 会话生命周期

```
ssh 客户端
  │ ① TCP → :2222
  ▼
otssh serve（gliderlabs/ssh，纯 Go，不带 OpenSSH 依赖）
  │ ② NoClientAuth：不校验密码/公钥，"用户名里的 token"就是唯一凭证
  │ ③ Parse(token) → HMAC 对不对 → 到期没到期 → 来源 IP 白名单 → 并发没超
  │ ④ --once 的话，原子地占用 nonce；已被用就直接拒
  │ ⑤ 起伪终端，exec 登录 shell（可按 --run-as 降权）
  ▼
  │ ⑥ time.AfterFunc(剩余有效期) → 到点 kill 整个进程组 → 关会话
  │ ⑦ 双向字节转发，可选落盘录制
  ▼
对方终端
```

第 ⑥ 步是重点：**校验发生在每次连接建立时，到期杀进程发生在会话进行中**。
很多方案只做了前者，于是"凭证过期了但人还挂在里面照用不误"。

---

## 命令参考

### `otssh init`

生成签名密钥（`secret`，32 字节）和 SSH 主机密钥（`host_ed25519`）。

| 参数 | 默认 | 说明 |
|---|---|---|
| `--dir` | `/etc/otssh` | 配置目录 |
| `--force` | | 覆盖已有密钥，**会让所有已签发凭证立即失效** |
| `--quiet` | | 只打印路径 |

### `otssh grant`

签发凭证。**不需要 serve 在跑**，甚至可以在一台离线机器上签。

| 参数 | 默认 | 说明 |
|---|---|---|
| `--ttl` | `10m` | 有效期，`30s` / `10m` / `2h` / `1h30m` 都行 |
| `--once` | 关 | 一次性：认证成功即作废，第二个连接必被拒 |
| `--fwd` | 关 | 允许 `ssh -L` 本地端口转发（`-R`/`-D` 不支持） |
| `--record` | 关 | 标记该会话需要录制（服务端要配 `--record-dir`） |
| `--host` | 自动探测 | 打印命令里用的地址；也可用 `$OTSSH_HOST` |
| `--port` | `2222` | |
| `-n` | `1` | 一次签几张 |
| `--json` | | JSON 输出，便于脚本消费 |
| `--token` | | 只输出 token 本身，最干净的脚本接口 |
| `--quiet` | | 只输出完整 ssh 命令 |
| `--ssh-config` | | 额外打印 `~/.ssh/config` 片段 |

`--host` 的取值顺序：`--host` → `$OTSSH_HOST` → `<dir>/endpoint`（`serve` 启动时写入）→ 自动探测本机出口 IP。

### `otssh serve`

| 参数 | 默认 | 说明 |
|---|---|---|
| `--listen` | `:2222` | 建议用独立端口，别去抢 sshd 的 22 |
| `--run-as` | 当前用户 | 会话以哪个系统用户运行；以 root 起动时通常是 `root` |
| `--shell` | 目标用户的登录 shell | |
| `--max-ttl` | `24h` | 超过这个长度的凭证直接拒登（兜底，防手滑签出 10 年） |
| `--idle` | `0`（关） | 空闲超时；有输入或有输出都算"不空闲" |
| `--max-sessions` | `32` | 并发会话上限 |
| `--allow-from` | 不限 | 来源 IP 白名单，逗号分隔 CIDR |
| `--audit` | `<dir>/audit.jsonl` | JSONL 审计日志；设 `off` 关闭 |
| `--record-dir` | 关 | 会话录制目录（记录服务端输出） |
| `--motd` | | 会话建立时显示的提示语 |
| `--advertise` | 自动探测 | 写进 `<dir>/endpoint` 的对外地址 |
| `--quiet-audit` | | 不把审计事件回显到 stderr |

### `otssh inspect <token>`

离线查看凭证内容，不连服务器。`--json` 输出结构化结果，退出码表示有效/无效——适合塞进脚本做巡检。

---

## 部署到 Linux

### 推荐：自包含发布包，一条命令装完

**目标机不需要 Go、不需要联网**——包里已经带了两个架构的静态二进制。

```bash
# 开发机上打包
make package                          # 产出 otssh-0.1.0-linux.zip

# 传到服务器，解压 + 安装，就这一条
scp otssh-0.1.0-linux.zip root@host:/root/
ssh root@host 'cd /root && unzip -q otssh-0.1.0-linux.zip && sudo bash otssh-0.1.0/install.sh'
```

装完之后 **zip 和解压目录都可以直接删掉**，功能不受影响：

```bash
rm -rf /root/otssh-0.1.0 /root/otssh-0.1.0-linux.zip
```

安装脚本依次做这些事：识别架构挑对应二进制 → 装到 `/usr/local/bin/otssh`（复制，不是软链）
→ `init` 到 `/etc/otssh` → 生成并启用 systemd unit（没有 systemd 就退回 `/etc/init.d`）
→ 放行防火墙端口 → **用真 ssh 客户端做一次完整登录自检** → 打印你的第一条 `grant` 命令。

常用参数（完整列表 `bash install.sh --help`）：

```bash
sudo bash install.sh --port 2200                       # 指定端口
sudo bash install.sh --allow-from 203.0.113.0/24       # 只允许指定网段
sudo bash install.sh --max-ttl 1h --max-sessions 8
sudo bash install.sh --record-dir /var/log/otssh/rec   # 开启会话录制
sudo bash install.sh --no-firewall --no-selftest
sudo bash install.sh --print-unit                      # 只看会生成的 unit，不安装
```

重复执行是安全的升级：会保留 `/etc/otssh` 里的密钥，已签发的凭证不会失效。

卸载（可重复执行，默认保留密钥和日志）：

```bash
sudo bash /usr/local/share/otssh/install.sh --uninstall
sudo bash /usr/local/share/otssh/install.sh --uninstall --purge   # 连配置与日志一起删
```

> 脚本副本会被放到 `/usr/local/share/otssh/`，所以删掉 zip 之后卸载入口依然在。

### 手动部署

不想用脚本的话：

```bash
make dist
scp dist/otssh-linux-amd64 root@host:/usr/local/bin/otssh
ssh root@host 'chmod +x /usr/local/bin/otssh && otssh init --dir /etc/otssh'
ssh root@host 'otssh serve --listen :2222 --run-as root'     # 先前台跑起来看看
# 想要 systemd：bash install.sh --print-unit 可以看到脚本会生成什么，照着改
```

### systemd unit 要点

安装脚本生成的 unit 里有两处值得注意：

```ini
KillMode=control-group     # 停服务时按 cgroup 把所有人的会话一起清掉，不留孤儿 shell
ProtectSystem=full         # /usr 只读；/etc /var /home 仍可写
ProtectHome=no             # 会话要能进 /root、/home，所以关掉
PrivateTmp=no              # 让会话看到系统 /tmp，避免"文件去哪了"的困惑
```

其余加固项见 `bash install.sh --print-unit` 的输出，每项都注明了会影响什么。

### 端口与网络

- 建议监听 `2222`，**不要动 22**——把系统 sshd 留着当后门，万一网关出问题你还能进去。
- 公网暴露时优先用 `--allow-from` 而不是只靠端口藏匿。
- 想跑在 443 上穿透封锁？otssh 是裸 SSH，不套 TLS，需要的话在前面放 `stunnel`/`sslh` 或 `socat`。

---

## 安全模型

### 一句话

> **token 就是一枚 bearer token。谁拿到那串字符串，谁就能以你指定的身份操作这台机器，直到到期。**

这是设计意图（"谁获得就可以控制"，正是你要的），但安全边界只有三条：**TTL、`--once`、`--allow-from`**。所以：

### 必须做的

1. **TTL 给短。** 5 分钟够用就别给 2 小时。`--max-ttl` 兜底设成 1h 或更低。
2. **敏感场景一律加 `--once`。** 凭证用过即废，被转发出去也没用。
3. **公网暴露一定配 `--allow-from`。** 例：`--allow-from 203.0.113.7/32`。
4. **主机指纹带外确认。** 第一次连接会问 `yes/no`，如果对方从不安全的渠道拿到命令，中间人可以冒充网关。把 `otssh init` 打印的 `SHA256:...` 用另一个渠道发过去。
5. **`secret` 文件当私钥对待。** 它是 0600，别提交进 git，别贴群里。泄露 = 任何人都能自签凭证。轮换方式：`otssh init --force`（旧凭证全部立即失效）。

### token 会漏到哪里

| 位置 | 情况 |
|---|---|
| 对方的 shell history | 他 `ssh ot1...@host` 那行会存进去 |
| 你的终端回滚缓冲 | `grant` 的输出 |
| 进程列表 | `grant --token` 的输出会出现在管道里 |
| 审计日志 | **otssh 有意只记 nonce（前 8 位）和到期时间，不记完整 token** |
| 系统 auth.log | 如果这个端口前面还有别的 sshd/bastion 记录用户名，会记全 |

结论：**别把 TTL 设长，别重复使用同一个 token。**

### 本版本刻意不做的事

- **`ssh -R` 反向转发**：gliderlabs 的 `ForwardedTCPHandler` 只拿得到 `ctx` 拿不到连接，
  无法按 token 鉴权，所以直接不注册——`-R` 会失败。少一个内网穿透入口不是坏事。
- **不监听 22**、不接管系统 sshd、不改 PAM。otssh 完全独立，删掉它不影响你原有登录方式。

### 没做的（自己补）

- **没有限流/防爆破**：公网长期暴露建议前面加 `fail2ban`，或用 nftables 限连接速率。
  （其实 token 有 80 bit HMAC + 8 位随机数，暴力猜不可行；风险主要是连接数打满 goroutine。）
- **没有多因素**：token 即全部。要更强就上 Teleport / step-ca 那套证书体系。

---

## 和现成方案比

写之前我把 GitHub 上相关的都过了一遍。**没有"一条命令生成限时凭证、谁拿到谁能控制这台机器 X 分钟"的轻量自建工具**，最接近的都在某一条上缺角：

| 项目 | 形态 | 差在哪 |
|---|---|---|
| **[tmate](https://github.com/tmate-io/tmate)** | `tmate -F` 打印 `ssh <token>@tmate.io`，零代码 | 默认托管在 tmate.io（可自建 tmate-ssh-server）；拿到 token 是进你的 **tmux**（能看到你原有窗口和历史），不是独立身份；**没有内建到期强断**，得自己 `timeout 600 tmate -F` 包一层 |
| **[upterm](https://github.com/owenthereal/upterm)** | `upterm host -- bash` 打印 `ssh <token>@uptermd.upterm.dev` | 同上思路，支持 `--read-only` / `--force-command` / 自建 uptermd；**同样没有内建 TTL** |
| **[sshpiper](https://github.com/tg123/sshpiper)** | 反向 SSH 代理，按用户名路由到后端 | 解决"多后端路由"，**不发凭证、不管时长**；要限时得自己写 plugin + 计时器 |
| **[sish](https://github.com/antoniomika/sish)** | 自建版 serveo，隧道 + 远程 shell | 同上，无凭证签发与 TTL |
| **[Teleport](https://github.com/gravitational/teleport)** | 完整堡垒机，短时效证书 + `max_session_ttl` + 录屏 | 功能最全，但**要部署 auth/proxy 一整套**；"一键发凭证"要走它的 invite/join 流程，不是一条命令 |
| **[Boundary](https://github.com/hashicorp/boundary)** | 会话有 max duration，**到点强断** | 重量级，需要控制平面 + worker |
| **Vault SSH CA** | `vault write ssh-client-signer/sign/role ttl=10m` | 签发很漂亮，**会话终止要自己接**；且你得先有 Vault |
| **[warpgate](https://github.com/Eugeny/warpgate)** | Rust 单二进制 SSH/HTTP 堡垒机，带录屏 | 凭证走它自己的账号体系，不是"发一条命令" |
| **JumpServer / NextTerminal** | 国产堡垒机 | 平台形态，部署成本高 |
| **Codespaces / Gitpod / cnb.space** | 就是你要的那种 `ssh <长token>@域名` | **是平台内建能力，不能自建到你自己的机器上** |

结论：要么重（Teleport/Boundary/Vault），要么缺 TTL（tmate/upterm），要么只做路由（sshpiper/sish）。
**otssh 补的就是这个缺口**：单二进制、无依赖、无状态凭证、发一条命令、到点强断。

如果你只是偶尔用一次、也不在乎 token 托管在别人那里，`timeout 600 tmate -F` 是最省事的；
要可控、要审计、要自己机器上跑，用 otssh。

---

## 测试

```bash
make test        # 单元测试 16 个用例：token 篡改/过期/跨密钥/编码边界/整数溢出
make e2e         # Windows 端到端 29 项断言（真 ssh 客户端，管道模式）
make e2e-linux   # Linux 端到端 30 项断言（真 ssh 客户端，伪终端 + 进程组强杀）
```

Windows 上想顺便验一遍 Linux 路径，不用装虚拟机：

```bash
make dist
wsl -- bash /mnt/d/Dsh_Work/otssh/test/wsl-verify.sh
```

（实测环境：WSL2 / Ubuntu 24.04.3，30/30 通过。）

端到端覆盖的内容（都是拿真 `ssh` 客户端打真服务端）：

- 正常登录、远端命令执行、倒计时横幅
- 篡改任意一位被拒 / 换成别人的密钥签的被拒 / 全 0 token 被拒
- 过期 token 被拒，未到期可用
- `--once` 第一次可用、第二次被拒
- **时间一到强断正在跑的长任务**（管道模式 + 伪终端模式各测一遍），并验证断开时刻 ≈ 剩余有效期
- `--max-ttl` 兜底拒绝超长凭证
- `inspect` 有效性判定与 id 一致性
- 审计日志落盘、含 start/deny、且不含完整 token
- `-L` 端口转发：带 `--fwd` 通、不带则被拒

跨平台：Windows 上可以开发调试（自动退化为管道模式），生产目标平台是 Linux。
`linux/amd64` 和 `linux/arm64` 都验证过交叉编译（约 4 MB，`CGO_ENABLED=0` 静态链接）。

### 这套测试抓到的两个真 bug

留个记录，也解释代码里那两处"看起来多余"的防御为什么必须留着：

1. **`exec.Wait()` 死锁。** 把非 `*os.File` 的 reader 赋给 `cmd.Stdin` 时，Go 会另起一个拷贝协程，
   而 `cmd.Wait()` 要等它结束；可那个协程又在等客户端关 stdin。
   结果就是"远端命令早就跑完了，连接却一直挂着"。修法：改用 `cmd.StdinPipe()`。
2. **服务自杀。** 非 root 运行时子进程没有独立进程组，`kill(-pgid)` 打到了 otssh 自己所在的组，
   到点强断的瞬间把服务、乃至启动它的 shell 一起杀了。Windows 上因为只杀单进程所以完全没暴露。
   修法：无条件给子进程 `Setsid`，并在 `killTree` 里加一道"pgid 等于自己就只杀单进程"的保险。


---

## 已知限制

- **单机网关。** token 只对持有同一 `secret` 的 `serve` 有效。想做成"一个网关管多台后端"（像 cnb.space 那样按用户名路由到不同容器），需要在 `run()` 里加一层后端选择——那正是 sshpiper 在干的事。
- **`--once` 不跨实例。** 多实例共享 `secret` 时重复使用只会被各自实例拒绝，不会全局拒一次。多实例要严格一次性，把 `usedSet` 换成 Redis。
- **`sftp` 子系统不支持。** gliderlabs 默认没有注册子系统处理器。`scp` 可以（走 exec 通道），`rsync --rsh ssh` 也可以。
- **`-R`/`-D` 不支持**，原因见[安全模型](#本版本刻意不做的事)。
- **录制是原始字节流**，不是完整 typescript 回放；只记录服务端输出，**不记录客户端按键**（避免录到密码）。文件开头有元信息注释。
- **Windows 上没有伪终端**，交互式 shell 会降级（无行编辑/无颜色），`ssh host 命令` 完全正常。
- **没有优雅处理客户端断线重连**：连接断了就是断了，token 还在有效期内可以重连。

---

## 文件结构

```
otssh/
├── install.sh           一键安装 / 卸载 / --print-unit
├── main.go              CLI：init / grant / serve / inspect
├── token.go             凭证编解码与校验（核心，无外部依赖）
├── serve.go             SSH 网关、会话生命周期、审计、录制
├── util.go              密钥、主机密钥、CIDR、用户查找等
├── proc_unix.go         伪终端 + 进程组强杀 + 降权（Linux/macOS）
├── proc_windows.go      管道模式降级实现
├── token_test.go        凭证单元测试
├── util_test.go         边界回归测试
├── Makefile             build / test / e2e / dist / package / install
├── LICENSE
├── dist/                交叉编译产物（make dist 生成，随发布包分发）
└── test/
    ├── e2e.sh           Linux 端到端
    ├── e2e.ps1          Windows 端到端
    └── wsl-verify.sh    在 WSL 里跑 Linux 端到端
```

发布包（`make package`）的结构：

```
otssh-0.1.0/
├── install.sh                     ← 入口，sudo bash install.sh
├── README.md  LICENSE  Makefile
├── dist/
│   ├── otssh-linux-amd64          静态二进制，目标机直接可用
│   └── otssh-linux-arm64
├── *.go  go.mod  go.sum           源码（想自己重编也有）
└── test/                          端到端脚本
```

依赖只有三个：`gliderlabs/ssh`（SSH 服务端框架）、`creack/pty`（伪终端）、`golang.org/x/crypto`。
`go.mod` 锁在 `go 1.22` / `x/crypto v0.31.0`，确保较老的 Go 也能直接编。

---

## License

MIT
