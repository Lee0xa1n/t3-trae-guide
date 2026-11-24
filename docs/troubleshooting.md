# Windows 上接 DSH / T3 / Trae 的排障手册

验证环境：Windows；T3 Code 0.1.11；DeepSeek Harness 桌面端（acp profile）；Trae CN 3.4.22；记录日期 2026-11-23，Trae 结论更新于 2026-11-24。

## 这本手册帮你解决什么

你在 Windows 上把 DeepSeek Harness（DSH）、T3 Code、Trae 这类 AI 工具接起来的时候，会反复撞上同一类问题：Electron 应用一点就闪退、启动器 spawn 不了批处理、ACP 协议格式报错、默认路由没 Key、第三方插件的安装脚本报错、服务重启就丢、局域网死活连不上、安装器静默失败、NSIS 写临时文件失败、抓 GitHub release 抓不到、push 抖一下就断。

这本手册是 2026-11-23 一天内实测的记录。第一把任务塞得太多，执行方反复试错烧了 50M+ tokens；拆小之后后面的任务一次过。所以手册的第一条原则就是拆任务——先读原则，再按案例查。

用法：遇到问题，先按“问题现象”对上案例，再按“怎么定位”里的报错原文和诊断指纹确认是不是同一个坑，最后照“怎么解决”做，做完按“怎么验证”确认。报错原文和诊断指纹是精华，定位时先对指纹。

诚实说明：每条都标了验证状态。标了“未验证”的，是当时没条件验证的，别当成定论。命令、界面、默认行为会随版本变，以各项目当时的官方说明为准。

## 通用原则

1. **一次只做一个任务，拆到自然的决策点**。一个任务里别塞两件不相干的事；做到一个能验证的节点就停，等下一步。
2. **每一步都要有个看得见的验证**。窗口打开了、接口返回 200、服务日志显示正常——没验证通过不往下走。
3. **报错原文记下来，试错别超过两次**。把报错原文写下来再想下一步，不要自己反复试，试错最烧时间。
4. **先查默认值**。默认路由、默认监听地址、默认密钥，逐项确认能不能用，不能直接信。
5. **密钥、token、密码绝不进仓库**。新密钥生成后只在对话里告诉本人。
6. **服务要进自动启动**。不要长期跑手动起的独立进程，重启就丢。
7. **从别的程序会话里启动程序，先看环境变量**。会话继承的环境变量会把新程序带偏。

## 先说结论：Trae 积分的三条路（2026-11-24 已验证）

想让 T3 烧 Trae 积分，三条路试过，两条死路，一条通路：

- **死路一：dsh-trae-api 代理。** dsh-trae-api 只是跑在 9220 端口的 HTTP 代理，把 Trae 积分转成 OpenAI/Anthropic 兼容接口。接口本身是通的，`/v1/models` 能返回 15 个模型。但它不向 DSH 注册任何模型，DSH 的 acp 模型选择器里根本看不到 Trae 的模型，这条路到 DSH 这里就断了。
- **死路二：T3 直连 Trae 接口。** 在 T3 里加 Claude provider，指向 9220 端口，有两个硬卡点，各自都能堵死：T3 的 Claude provider 必须启动本机 `claude` CLI，报错 `Claude Agent CLI (claude) was not found on PATH`；即使绕过这一步，9220 的 `/v1/messages` 返回 200 但 `content` 为空、`output_tokens` 为 0，不出字。
- **通路：our free model 插件。** dsh-our-free-model（Ebony-Vinyl/dsh-our-free-model）是真正的 LLM provider 插件，内置 TRAE 账号渠道。装进 acp 后出现 TRAE 组 40+ 模型，默认路由改成 trae / DeepSeek-V4-Flash，ACP 和 T3 两边都验证出字，token-ledger 确认 Trae 在烧。

完整走通的教程在这里：https://github.com/Lee0xa1n/t3-trae-guide

下面关于 dsh-trae-api 的三个案例（setup.js 报错、服务重启就丢、局域网三件套）讲的是“怎么把这个代理跑起来”，本身没错；但在新结论下要明白：代理跑起来了，也接不进 DSH 的模型体系。想烧 Trae 积分，走上面那条通路。

dsh-trae-api 是第三方插件（学习研究用，请遵守 Trae 的服务条款，账号风险自担）。

## 案例一：Electron 应用从别的程序会话启动就静默闪退

**问题现象：** T3 Code 第一次启动，exit 0、无日志、不生成 userData。

**怎么定位：** `--version` 打印出 Node 版本号（v24.22.1）而不是应用版本；`--help` 打印 Node 的 help；`--no-sandbox` 报 `bad option: --no-sandbox`——这说明应用在 Node 模式下运行。`--enable-logging`、`ELECTRON_ENABLE_LOGGING`、`ELECTRON_ENABLE_STACK_DUMPING` 全试过，没有任何日志；事件日志、CrashDumps、WER 里也没有崩溃记录。根因：DSH 会话里的 `ELECTRON_RUN_AS_NODE=1` 被继承；反编译 main.cjs 发现 `requestSingleInstanceLock` 失败会静默 `quit()`。

**怎么解决：** 启动前清除这个环境变量。

**怎么验证：** 重新启动，应用正常打开；`--version` 打印应用版本而不是 Node 版本。

**以后：** 从任何程序的会话里启动 Electron 应用，都先检查继承的环境变量。

## 案例二：T3 的 Local ACP 里填 `dsh` 跑不起来

**问题现象：** provider 配了 `dsh --profile acp`，启动失败。

**怎么定位：** 报错原文：

```
Local ACP executable is not available on this environment's PATH.
Local ACP commands launch without a shell. For a Windows batch wrapper, configure the underlying executable (for example node.exe) and pass the script path as an argument.
```

根因：T3 直接 spawn 可执行文件，不走 shell，`dsh.cmd` 批处理没法直接运行。

**怎么解决：** 可执行文件填 DeepSeek Harness.exe 的绝对路径，参数每行一个：

```
--expose-internals
<DSH 安装目录>\resources\app.asar\dsh\node_modules\@deepseek-ai\dsh-desktop-host\lib\cli.js
--profile
acp
```

环境变量加 `ELECTRON_RUN_AS_NODE=1`。

**怎么验证：** T3 里 provider 能启动，ACP 会话能正常出字。

**以后：** 凡是“直接 spawn、不走 shell”的启动器，都按这个思路填：exe 绝对路径 + 内部启动参数。

## 案例三：ACP 协议对接的五个固定坑

**问题现象：** 按文档对接 ACP，报各种格式错误。

**怎么定位、怎么解决：** 对接之前先按这五条自查：

1. node 子进程里 spawn `dsh` 报 ENOENT：dsh 在系统里是 `dsh.cmd`，批处理不能直接 spawn；改成直接 spawn exe。
2. authenticate 的 `params.token` 必须是字符串（服务器其实忽略它，填 `test` 就能过）。
3. `session/new` 的 `mcpServers` 字段必填，没有就传 `[]`。
4. `prompt` 字段必须是数组 `[{type:'text', text:'...'}]`，传字符串会报 `Invalid input: expected array`。
5. `set_config_option` 的参数名是 `configId`，不是 `optionId`。

**怎么验证：** 逐项对过之后，`session/new` 和 `prompt` 能正常返回，不再报格式错误。

**以后：** 对接 ACP 先按这五条自查，别在协议格式上烧时间。

## 案例四：acp 默认模型路由没 Key

**问题现象：** DSH ACP 通了，但默认模型调不通。

**怎么定位：** 报错原文：

```
Internal error: turn failed: llm-deepseek: no API key for provider route "deepseek-official"; store DEEPSEEK_API_KEY through the credentials service...
```

根因：acp 默认路由 deepseek-official 没有 API Key。

**怎么解决：** 会话里切路由。想烧 Trae 积分就切到 trae 路由（见上面“先说结论”）；只用 DeepSeek 账号额度就切到 deepseek-account 路由。

**怎么验证：** 切完路由发一句话，能正常出字。

**以后：** 接好通道先确认默认模型能不能出字，不行就换路由、换凭据，别在通道上浪费时间。

## 案例五：dsh-trae-api 的 setup.js 报错 `require is not defined`

**问题现象：** 插件自带 setup.js 一跑就报 `ReferenceError: require is not defined in ES module scope`。

**怎么定位：** package.json 里 `"type": "module"`，Node 强制按 ESM 解析，里面的 require 失效。

**怎么解决：** 复用插件 `src/trae-decrypt.js` 的 CommonJS 解密逻辑，拷贝成 `.cjs` 后缀手动解密，写进 `.env`。

**怎么验证：** 解密成功，`.env` 里拿到 token，服务能启动，`/v1/models` 能返回模型列表。

**在新结论下的位置：** 这一步只是让代理跑起来。代理本身能跑，但它不向 DSH 注册模型，所以就算 setup.js 修好了，acp 里还是选不到 Trae 模型。想烧 Trae 积分，走 our free model 插件那条路（见上面“先说结论”）。

**以后：** 第三方插件的安装脚本出问题，先读它的源码找现成逻辑，别从头写。

## 案例六：服务手动跑着，重启就丢

**问题现象：** dsh-trae-api 一开始是独立进程跑的，DSH 重启后 9220 就没了。

**怎么定位：** 第一次重启 dsh web profile 时，PowerShell 的 `-ArgumentList` 把含空格的 cli.js 路径按空格拆了，报 `Cannot find module`（路径被截断）；改 `cmd /c` 启动又卡住等管道 EOF。

**怎么解决：** 停掉独立进程，用 Start-Process 把 stdout/stderr 重定向到日志文件（不读管道），启动 `dsh --profile web --no-open`，插件自动拉起服务。

**怎么验证：** 重启 DSH 后，9220 端口还在，接口能返回 200。

**以后：** 含空格的路径别用 PowerShell `-ArgumentList` 传；启动后台进程输出写文件，别读管道。装完先确认服务是“随启动自动拉起”还是“手动跑的”，手动的要改成自动的。

## 案例七：局域网访问要三件套

**问题现象：** 本机 `127.0.0.1:9220` 通，局域网里的设备连不上。

**怎么定位、怎么解决：** 三件套，缺一不可：

1. `.env` 里 `HOST=0.0.0.0`（只监听本机改成监听所有网卡）；
2. 默认 API Key 是公开的，必须换成新生成的随机密钥；
3. Windows 防火墙弹窗要点允许。

生成密钥别用 PowerShell：`[System.Security.Cryptography.RandomNumberGenerator]::Fill()` 在有的机器上不存在（报“不能调用方法”），生成的是空值；改用 `node -e "console.log(require('node:crypto').randomBytes(32).toString('hex'))"`。

**注意：** 自定义 API Key 丢过一次——换账号时发现 `.env` 里的 API_KEY 变回了默认值 `trae-local-api`（公开的），不知道哪次操作丢的，已恢复成自定义 key。以后每次动 .env、重启服务后，顺手验证一下 key 还是不是自定义的。

**怎么验证：** 局域网设备用新 key 能调通接口；用完改回 127.0.0.1 之后，局域网连不上、本机还能通。

**未验证：** 本机自环验证不受防火墙影响；“局域网设备连不上→查防火墙”这条当时没实测，留待第一次局域网连通时再验证。

**以后：** 局域网暴露随用随收，用完改回 127.0.0.1。

## 案例八：T3 网页 UI 自动化的几个小坑

**问题现象：** 用脚本操作 T3 网页 UI 时，配对和输入框出问题。

**怎么定位、怎么解决：**

- `t3 pair` 生成的配对 token 只有 5 分钟 TTL，每次新浏览器会话都要重新配对。
- headless 下 `input.select()` 全选不可靠，值会残留成旧路径片段；改成 click 聚焦 → Ctrl+A → 删除 → 输入。

**未验证：** 当时通过哪个按钮组合进到 Local ACP 表单（Show more → Local ACP command），操作的人自己也记不清，没再重演。写下来是提醒你别在这上面死磕，先按界面实际的样子找。

**怎么验证：** 配对成功、输入框里的值和预期一致。

## 案例九：安装器静默参数别默认能用

**问题现象：** 某个 Tauri 2 打包的 NSIS 安装器，加 `--help` / `/S`，无输出，30 秒后进程消失；普通双击启动，进程 4-8 秒后消失，无窗口、无安装痕迹。

**怎么定位：** 静默参数没被识别（具体内部机制记不清了）；普通启动消失的确切原因没深究，候选：没提权、UI 会话限制、安全组件拦截。

**怎么解决：** 放弃静默，改交互式 + 管理员运行（`-Verb RunAs`）。

**怎么验证：** 管理员交互安装能正常弹出安装向导并装完。

**以后：** 安装器没反应先别怀疑包坏了，先试提权交互安装。

## 案例十：NSIS Error 写临时文件失败

**问题现象：** 提权启动后弹 `NSIS Error / Error writing temporary file. Make sure your temp folder is valid.`

**怎么定位：** TEMP 目录本身正常（测试写入成功），怀疑安全软件拦截：机器上的 EDR / Defender 可能会拦未知发布者安装器写 %TEMP%（推断，安装器没给更细的日志）。

**怎么解决：** 安装包复制到桌面干净 ASCII 路径 + 新建独立临时目录（纯 ASCII 路径，例如 `C:\app-temp`），进程级 TEMP/TMP 指向它 + 管理员运行 → 临时目录出现 nse*.tmp，安装成功。

**怎么验证：** 临时目录里出现 nse*.tmp，安装完成。

**以后：** 报写临时文件失败，先验 TEMP 目录本身，再怀疑安全拦截，换干净路径 + 独立临时目录 + 提权三件套。

## 案例十一：抓 GitHub release 的工具组合

**问题现象：** `web_fetch` 抓 release 页失败 `fetch failed`（不走本机代理）；`Invoke-RestMethod` 调 API 返回 403；`Invoke-WebRequest` 报 PowerShell 非交互模式限制。

**怎么定位：** 三个工具各自的短板，不是一处能修好的。

**怎么解决：** 改用 `curl.exe` + `-x` 指定本机代理地址，先抓 HTML 确认版本，再调 API 拿 asset。

**怎么验证：** 拿到正确的 release 版本和 asset 链接；下载安装包后做 SHA256 校验，对上官方值再装。

**以后：** 下载安装包一定做 SHA256 校验，对上官方值再装。

## 案例十二：push 遇网络抖动

**问题现象：** push 时 `Recv failure: Connection was reset` / `schannel: failed to receive handshake`。

**怎么定位：** 网络抖动，不是仓库或凭据问题。

**怎么解决：** 走代理重试 + rebase 合并远端后再推。

**怎么验证：** push 成功，远端有这次提交。

**以后：** push 失败先别慌，换代理重试、rebase 一下。

## 案例十三：卸载要确认干净，通用组件别顺手删

**问题现象：** 软件卸了，但配置、日志、临时目录还留在机器上；或者卸载时顺手删了别的软件也在用的通用组件。

**怎么定位、怎么解决：**

- 官方卸载程序（/S）+ 手动删配置、日志、安装包、排障用的临时目录 + 在用户目录按软件名递归搜关键词，确认无残留。
- 通用组件别顺手删：比如 Apple Mobile Device Support 是苹果的通用 USB 连接组件，多个苹果软件都可能依赖它，卸载别的软件时别顺手删掉，列出来自己决定。

**怎么验证：** 关键词搜索无残留；通用组件还在，被依赖的软件能正常工作。

## 换账号 / 换软件时重走的清单

1. 装软件 → 2. 登录（自己输账号密码）→ 3. 取凭据（token/key，确认有效期）→ 4. 检查默认值（路由、监听地址、密钥）→ 5. 改成自动启动 → 6. 逐项验证（窗口、接口、日志）→ 7. 报错原文写回，等下一步。
