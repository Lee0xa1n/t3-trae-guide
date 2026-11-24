# 让 T3 Code 烧 Trae 积分：把免费积分变成手机上的编程助手

验证环境：Windows；T3 Code 0.1.11；DeepSeek Harness 桌面端（acp profile）；Trae CN 3.4.22；our free model 插件 2.9.0；验证日期 2026-11-24。

配套排障手册：[Windows 上接 DSH / T3 / Trae 的排障手册](docs/troubleshooting.md)

## 这篇教程能帮你做到什么

T3 Code 是一个开源的编程控制台：电脑上跑服务，手机 App 当遥控器，随时随地指挥电脑上的 AI 干活。但 T3 自己没有模型，烧的是它接的那个 agent 的额度。

如果你已经在用 DeepSeek Harness（DSH），又手里有 Trae 的免费积分，这篇教程帮你把三者接起来：**T3 → DSH → Trae 积分**。接通之后，你在手机 T3 里发一句话，电脑上的 DSH 用 Trae 积分里的模型回你，不用另外花钱买 DeepSeek 官方 API。

适合谁：

- 已经在用或打算用 T3 Code + DeepSeek Harness 的人；
- 有 Trae 账号、想把积分花在手机遥控编程上的人；
- Windows 用户（下面的坑都是 Windows 上实测的）。

不适合谁：只想在 Trae 自己的 IDE 里用积分的人——那不需要这篇教程，直接在 Trae 里用就行。

## 为什么值得折腾：先算一笔账

Trae CN 的免费积分大致是这样（以 Trae 官方规则为准，随时可能变）：每月登录送 500，新用户一次性 1500，每天手动签到 100，每批积分有效期 31 天。坚持签到，一个月大约能攒 3000 积分。Flash 系列模型最便宜。

DeepSeek 官方账号的免费额度用完就要付费买 API。把 Trae 积分接进 T3，等于多了一条免费的燃料管线。代价是一次性半小时的接线工作，和下面写明的两个 Windows 坑。

## 准备工作：三个软件

| 软件 | 从哪下 | 版本（验证时） |
| --- | --- | --- |
| DeepSeek Harness | https://www.deepseek.com/download/（Windows 64 位） | 桌面端 |
| Trae CN | https://www.trae.cn（国内版，下载页 https://www.trae.cn/ide/download） | 3.4.22 |
| T3 Code | https://github.com/pingdotgg/t3code，或 winget 安装（见步骤六） | 0.1.11 |

另外需要一个 our free model 插件（Ebony-Vinyl/dsh-our-free-model），它是这条通路的关键，见步骤三。

先把三个软件都装好：DSH 装完打开一次，Trae CN 登录自己的账号（国内版和国际版账号不互通，用国内版），T3 装完能正常打开。这三个都能独立跑通，再往下接。

## 步骤一：确认 DSH 的 acp 能用

打开 DSH，确认 acp profile 可以正常对话。注意一点：acp 默认路由 deepseek-official 没有 API Key，默认模型调不通是正常的，先切到 deepseek-account 路由，用 DeepSeek 账号验证一次能出字。

这一步的意义是先证明 DSH → T3 这条底座是通的，后面再换燃料。

## 步骤二：在 Trae 里确认积分

打开 Trae CN，确认账号已登录，积分余额大于 0。每天签到领 100 积分的入口在 Trae 应用内，顺手领一下。积分是按账号算的，之后 DSH 里用的就是这个账号的积分。

## 步骤三：装 our free model 插件

这是唯一走通的通路，装法有讲究，先看警告。

### 警告：别装 npm 上那个裸名的包

npm 上有一个裸名的 `dsh-our-free-model`，**不是这个项目发布的**。仓库 README 明确写了：那个包是第三方打包的，`index.js`、`client.js` 被改动过，还多了仓库从未有过的 `src/channel-visibility.js`。装它得不到审核过的构建，也拿不到签名清单校验。

本仓库自己发布的是带 scope 的 **`@ebony-vinyl/dsh-our-free-model`**，与 Releases 里的 `dsh-our-free-model-<版本>.tgz` 同源。只认这个。

### 桌面端：用应用内插件管理器装

如果你用的是 DeepSeek Harness 桌面端（Electron），**命令行装不上**。桌面端的 profile 只允许桌面应用自己管理，命令行 `dsh plugin add` 会报错：

```
profile "desktop" is managed exclusively by the Electron application
```

做法：打开 DSH → 设置 → 插件，在插件管理器里搜索并安装 our free model。装完重启一次应用。

### 纯 web profile：用命令行装

如果你用的是 `dsh web` 起的纯 web 服务，可以用命令行：

```bash
dsh plugin --profile web add @ebony-vinyl/dsh-our-free-model@2.9.0
```

`--profile` 填你实际用的 profile 名；版本号以仓库 Releases 为准。装完重启一次应用。

### 本地安装（备用）

如果插件市场自动安装失败（比如本机 git 配了 `insteadOf` 重写导致 `git ls-remote` 失败），改用本地安装：从仓库 Releases 下载发布包解压，然后：

```bash
dsh plugin add <解压目录>
```

## 步骤四：在插件里登录 TRAE 渠道

our free model 是一个真正的 LLM provider 插件，内置了 10 个账号渠道，TRAE（字节）是其中之一，开箱即用。凭据只写进宿主的凭据库，不会明文出现在别处。

做法：在 DSH 的设置里打开 Our Free Model，找到渠道列表里的 TRAE，按提示用自己的 Trae 账号登录/授权。登录成功后，打开 DSH 的模型选择器，应该能看到 Our Free Model 分组，里面按渠道分组，TRAE 组有 40+ 个模型，包含 DeepSeek-V4-Flash。

如果 TRAE 组没出现：先确认插件已正确安装并重启过应用，再检查插件设置里的渠道列表有没有 TRAE；还不行就看插件的公告中心有没有相关说明。

## 步骤五：把 acp 默认路由改成 trae

在 DSH 的 acp 里，把默认路由改成 **trae / DeepSeek-V4-Flash**。然后在 ACP 里发一句话测试，确认能正常出字。

这一步做完，DSH 本身就已经在烧 Trae 积分了。T3 只是换了一个遥控器，燃料不变。

## 步骤六：把 T3 接到 DSH

### 装 T3

Windows 上用 winget 装：

```powershell
winget install T3Tools.T3Code
```

也可以从 GitHub Releases 下安装包。装完先正常打开一次，确认能进主界面。

### 在 T3 里配 Local ACP provider

在 T3 里添加一个 Local ACP provider，接 DSH。这里有两个 Windows 坑，都是实测踩出来的，官方文档没写全。

**坑一：`dsh` 命令填不进去**

T3 的 Local ACP 是直接 spawn 可执行文件，不走 shell。`dsh` 在 Windows 上是个 `.cmd` 批处理，没法直接运行，会报错：

```
Local ACP executable is not available on this environment's PATH.
Local ACP commands launch without a shell. For a Windows batch wrapper, configure the underlying executable (for example node.exe) and pass the script path as an argument.
```

做法：可执行文件填 DeepSeek Harness.exe 的绝对路径，参数每行一个：

```
--expose-internals
<DSH 安装目录>\resources\app.asar\dsh\node_modules\@deepseek-ai\dsh-desktop-host\lib\cli.js
--profile
acp
```

环境变量加 `ELECTRON_RUN_AS_NODE=1`。

**坑二：从 DSH 会话里启动 T3 会静默闪退**

如果你是从 DSH 的终端会话里启动 T3，T3 可能第一次启动就直接退出，exit 0，没有任何日志，也不生成用户数据。

诊断指纹：运行 `T3 Code --version`，如果打印出来的是 Node 的版本号而不是应用版本，或者 `--help` 打印的是 Node 的 help，说明应用跑在了 Node 模式下。

原因：DSH 会话里的 `ELECTRON_RUN_AS_NODE=1` 被 T3 继承了。

做法：启动 T3 之前先清除这个环境变量。以后从任何程序的会话里启动 Electron 应用，都先检查继承的环境变量。

### 验证 T3 能出字

在 T3 里新建一个会话，发一句话，确认能正常收到回复。回复走的是 DSH 的 acp，用的就是步骤五设好的 trae / DeepSeek-V4-Flash。

## 步骤七：手机上选一次模型

在手机上装 T3 App（iOS/Android 都有），用 T3 Connect 连上电脑。新建会话时，在模型选择器里选一次 **DeepSeek Harness / DeepSeek-V4-Flash**，之后 T3 会记住，不用每次都选。

注意：手机只是遥控器，干活的还是电脑。电脑关机、或者电脑上的 DSH 停了，手机就指挥不动。

## 验证：三条缺一不可

接通之后，按这三条逐一确认，缺一条都别当成通了：

1. DSH 的 acp 模型列表里有 TRAE 组，且能出字；
2. T3 里用这个模型能正常回复；
3. Trae 的用量记录（token-ledger）里能看到积分在扣。

只满足前两条、第三条看不到，说明烧的不是 Trae 的积分，先别当成通了。

## 别走的弯路：两条死路

这两条路我们都实测过，都死了，写出来是帮你省时间。

### 弯路一：dsh-trae-api 代理

dsh-trae-api（Wang-JQ77/dsh-trae-api）把 Trae 积分转成 OpenAI/Anthropic 兼容接口，跑在本地 9220 端口。接口本身是通的，`/v1/models` 能返回 15 个模型。

但它只是一个 HTTP 代理，不是一个 LLM provider 插件。它不向 DSH 注册任何模型，DSH 的 acp 模型选择器里根本看不到 Trae 的模型。所以这条路到 DSH 这里就断了。

补充：这个插件的默认 API Key 是公开的默认值 `trae-local-api`，如果用它，务必换成自己生成的随机密钥；只监听本机（127.0.0.1）就够用时，不要改成 0.0.0.0。

### 弯路二：T3 直连 Trae 接口

在 T3 里加一个 Claude provider，填环境变量 `ANTHROPIC_BASE_URL=http://localhost:9220/v1` 指向 dsh-trae-api。有两个硬卡点，各自都能堵死：

1. T3 的 Claude provider 架构上必须启动本机 `claude` CLI，环境变量是传给这个 CLI 的。本机没有装 claude CLI，直接报错：

   ```
   Claude Agent CLI (claude) was not found on PATH
   ```

2. 即使绕过这一步，9220 的 Anthropic 接口本身也有问题：实测 `POST /v1/messages` 返回 200，但 `content` 为空、`output_tokens` 为 0，不出字。截至 2026-11-24 没有修复。

## 常见问题

**Q：acp 里 deepseek-official 路由调不通？**

正常的。这个路由默认没有 API Key。验证 DSH 本身用 deepseek-account 路由；走 Trae 用 trae 路由。

**Q：插件自己的本地网关端口 8326 是什么？**

our free model 插件自带一个本地 OpenAI 网关，默认 `127.0.0.1:8326`，在设置 → Our Free Model → 本地转发里启用。它和 dsh-trae-api 的 9220 端口不是一回事，不要混用。

**Q：T3 的权限模式怎么选？**

T3 每个会话有权限模式（Supervised / Auto-accept edits / Auto / Full access）。刚接通先用 Supervised，熟悉了再放宽。电脑上的 DSH 能读写文件、执行命令，权限给出去之前先想清楚。

**Q：Trae 积分烧完了怎么办？**

DSH 的 acp 可以随时把默认路由切回 deepseek-account，改回 DeepSeek 账号额度。Trae 积分每天签到会续上，不用重装任何东西。

## 风险与有效期

- our free model 是第三方插件，不是 DeepSeek、Trae 或 T3 的官方组件。它的 TRAE 渠道是 2026 年 11 月的新功能，行为随时可能随版本变化；本指南只保证在上面写的验证环境和日期下走通过。
- dsh-trae-api 的文档自称仅供学习研究、遵守 Trae 使用条款。用第三方插件转接账号额度，账号风险自担。只用自己的账号，不要转卖接口——低价转卖境外大模型接口已有站长被刑事拘留的案例。
- Trae 积分的获取和扣费规则由 Trae 定，随时可能变。免费渠道、第三方中转，随时可能失效。
- 不要把任何 token、密码、API Key 写进仓库、聊天记录或公开的教程里。

## 参考链接

- T3 Code：https://github.com/pingdotgg/t3code
- DeepSeek Harness：https://www.deepseek.com/harness/（下载页 https://www.deepseek.com/download/；仓库 https://github.com/deepseek-ai/deepseek-harness）
- Trae CN：https://www.trae.cn（下载页 https://www.trae.cn/ide/download）
- our free model 插件：https://github.com/Ebony-Vinyl/dsh-our-free-model
- dsh-trae-api（弯路一提到的代理）：https://github.com/Wang-JQ77/dsh-trae-api
