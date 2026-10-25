# Cloud VNC Browser —— 用 GitHub Actions 开一台限时云端浏览器

手动触发一次 workflow，约 2~3 分钟后你会得到一个链接，点开就是一台云端 Chrome 的实时画面（可用鼠标键盘直接操作），到期（默认 30 分钟）自动销毁，也可以随时手动取消。

## 使用方法

1. 新建一个 GitHub 仓库（公共仓库免费跑；私有仓库会消耗 Actions 分钟数）。
2. 把 `cloud-browser.yml` 放到仓库的 `.github/workflows/` 目录下（必须在默认分支上，否则 Actions 页看不到）。
3. **配置 Secret**：仓库 **Settings → Secrets and variables → Actions → New repository secret**
   - Name：`FRP_TOKEN`
   - Value：你 `frpc.toml` 里 `metadatas.token` 的值
4. 打开仓库的 **Actions** 标签页 → 左侧选 **Cloud VNC Browser** → **Run workflow** → 填时长（默认 30 分钟）→ 运行。
5. 等 job 跑到「输出访问地址」这一步（约 2~3 分钟），在运行页面的 **Summary** 里（或该步骤日志里）找到：
   `http://frp.foxhank.top:随机端口/?autoconnect=1&password=xxxx`
   点开即用，密码已内置在链接里。
6. 想提前结束：在该 run 页面点 **Cancel workflow**，云端会话立刻销毁。

## 它是怎么工作的

```
GitHub runner（不允许外部直连）
 ├─ Docker: selenium/standalone-chrome（自带 Xvfb + VNC(5900) + noVNC(7900) + Chrome）
 ├─ frpc（0.65.0，与你本地同版本）→ frp.foxhank.top:7000
 │    type = "tcp"，remotePort = 随机端口（20000-60000，冲突自动换）
 │    映射：frp.foxhank.top:随机端口 → runner 本机 7900（noVNC）
 └─ sleep 计时 → 到期销毁整个 runner
```

访问链接里的随机端口每次触发都会变；token 走 GitHub Secrets，不会出现在代码和日志里。

## 配置项（yml 顶部 env 区）

| 变量 | 说明 |
|---|---|
| FRP_SERVER / FRP_SERVER_PORT | frp 服务端地址和端口，当前 `frp.foxhank.top:7000` |
| FRP_USER | frp 用户名，当前 `user67` |
| FRP_TOKEN | **Secret**，frp 认证 token |
| duration_minutes | 会话时长，1~35 分钟，默认 30 |
| 硬性上限 | job `timeout-minutes: 40` 兜底，超时强制销毁 |
| 并发 | 配置了 concurrency，同一时间只保留一个会话，重新触发会取消旧会话 |
| 分辨率 | 默认 1600×900，在 yml 里改 `SE_SCREEN_WIDTH / SE_SCREEN_HEIGHT` |

## 常见问题

- **提示端口不可用、重试 8 次都失败**：你的 frp 服务端可能限制了该账号可用的 remotePort 范围（很多免费 frp 面板只分配固定端口段）。把 yml 里 `shuf -i 20000-60000` 改成面板允许的范围即可。
- **frp 登录失败**：检查 Secret `FRP_TOKEN` 是否填对、`FRP_USER` 是否还是你账号对应的用户名。
- **想用原生 VNC 客户端（TigerVNC/RealVNC）连**：链接给的是 noVNC 网页版（7900 端口）。要客户端直连的话，在 yml 的 `[[proxies]]` 里再加一段，`localPort` 改成 5900，映射成功后用 `frp.foxhank.top:新端口` 连接。

## 安全须知（务必读）

- **链接 = 完全控制权**：随机端口 + 随机 8 位密码只是"不容易被猜到"，不是强安全，不要把链接发给别人。
- **明文传输**：frp 这条 TCP 隧道没有 TLS，流量是明文的，不要在云端浏览器里登录重要账号。
- **公开仓库的运行记录任何人可见**：Summary 里的访问链接会泄露。要么用私有仓库（扣 Actions 分钟数），要么跑完删掉运行记录。
- **GitHub ToS**：Actions 服务条款不建议用来跑与开发无关的常驻服务。临时、低频地开半小时一般没事，但风险自负，不要当免费 VPS 用。
- **费用**：公共仓库免费；私有仓库按分钟计费（Linux 为 1 倍率，每开一次约消耗 30+ 分钟额度）。

## 想改点别的

- **换 Firefox**：镜像改成 `selenium/standalone-firefox:latest`。
- **要整个 Linux 桌面（不只是浏览器）**：镜像换成 `accetto/ubuntu-vnc-xfce-g3`，noVNC 端口是 6901（默认密码 headless），把 frp 代理的 `localPort` 改成 6901，链接参数改成 `?autoconnect=1&password=headless`。
