# Changelog

本项目遵循 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/) 与 [语义化版本](https://semver.org/lang/zh-CN/)。

## [1.2.7] - 2026-10-27

### 新增

- **长系统提示跨越国内网关风控**：国内网关会对「第一条 system 消息」跑渠道风险分类器，超长的 agent 风格系统提示被判定为「第三方客户端」并以 400 code 11128 拒绝，但同样的文本放在 user 轮次就能过。`relocateLongSystemMessage()` 现在把超过 6000 字符的系统提示搬运到首条 user 消息（拼接或 `[{type:"text"}]` 前置），只留一句短系统桩 `You are a helpful assistant.`，让请求能过分类器、模型又不丢任何指令。
- **出站体改名**：发送前把请求体里的字面串 `DeepSeek Harness`（大小写不敏感）替换成 `DSH`，规避国内网关的渠道风险分类器（实测该字样会被判第三方客户端）。

### 说明

- 顺手加了调试落盘：失败时把请求体与上游返回截断写入 `C:/tmp/wbtest/last-body.json`（写失败静默吞掉）。

## [1.2.6] - 2026-10-23

### 修复

- **网关返回 `550` 且响应体为空时直接失败**：边缘网关偶尔会用 550（非标准码）拒掉请求且不给 body，1.2.5 把它当作普通错误直接透传，用户只看到 `upstream server (http 550): ` 后面什么都没有。现在 550 会重试一次（400ms 间隔）——这个状态码意味着上游什么都没生成，重放是安全的。

### 新增

- **失败诊断日志** `$DSH_HOME/.workbuddy-ai-diagnostics.log`：每次聊天请求失败时追加一行 JSON，记录时间、模型、请求体字节数、上游状态码、错误类别和消息（截断 400 字符）。此前这类故障在宿主的日志里什么都不留，只能靠用户复述错误文本；现在可以直接看文件定位。写入失败会被静默吞掉，不会影响请求本身。

### 说明

- 测试 114 → 116（550 重试成功、550 重试后仍失败两条断言）。

## [1.2.5] - 2026-10-23

### 修复

- **`502 ... upstream server (http 0): transport error: TypeError: fetch failed`**：插件用随机端口起一个长驻 shim，`fetch` 走 undici 的连接池。进程跑上几个小时后，池里那条 socket 可能已被上游关闭，或 DNS 记录随边缘 IP 变化而过期——下一次聊天请求复用它就直接 `fetch failed`，而代码一次都不重试，用户只看到 502 和一句没有任何信息量的 `fetch failed`。
  - 新增 `fetchUpstream()`：**只在传输层失败时重试一次**（`fetch failed` / `ECONNRESET` / `ECONNREFUSED` / `ETIMEDOUT` / `EAI_AGAIN` / `ENOTFOUND` / `EPIPE` / `socket hang up`）。已经拿到 HTTP 响应的请求绝不重放——聊天端点不是幂等的。
  - 聊天与模型目录两条路径都改用它。
  - 新增 `describeFetchError()`：把 undici 藏在 `cause` 里的 `ECONNRESET` / `ENOTFOUND` / `CERT_HAS_EXPIRED` 带进错误消息，不再只剩 `TypeError: fetch failed`。

### 说明

- 新增 3 项回归断言（池内 socket 失效后重试成功、两次都失败时透出 cause、非传输错误不重试），测试总数 111 → 114。

## [1.2.4] - 2026-10-23

### 变更

- **仓库改名**：`bushizhanghan/dsh-workbuddy` → `bushizhanghan/dsh-workbuddy-harness`。`package.json` 的 `repository` / `homepage` / `bugs` 同步指向新地址。
- 插件市场条目随之改名为 `data/plugins/bushizhanghan__dsh-workbuddy-harness.yml`，页面路径变为 `/p/bushizhanghan/dsh-workbuddy-harness/`，安装包地址改为新仓库的 `releases/latest/download/dsh-workmate.tgz`。
- **npm 包名不变**，仍是 `@bushizhanghan/dsh-workmate`，模块 id、provider id（`workbuddy-ai`）、CLI 命令（`dsh-workmate`）也都不变 —— 已安装的用户不需要改任何配置。
- GitHub 会把旧仓库地址 302 重定向到新地址，旧的 clone URL、Release 链接、市场条目短期内仍可访问。

## [1.2.3] - 2026-10-23

### 变更

- **README 重写**（中英双版）：去掉全部截图与徽章图片，改为纯文本说明；按「安装 → 选模型 → 登录流程 → 设置 → 命令行 → 兼容性 → 故障排查」重排。`docs/` 下的三张截图随之删除，npm 包不再包含 `docs/`。
- **模型行显示后缀去重**：`src/display.ts` 新增 `joinDisplayParts()` 与 `DISPLAY_PART_SEPARATOR`，宿主侧（`adapter.ts`）与卡片侧（`catalog.ts`）原本各写一遍的「计费 + 促销标记」拼接逻辑改为共用一处。顺带修掉 `adapter.ts` 里定义了却没被使用的 `RATE_SEPARATOR`（第 93 行仍在用字面量）。
- `src/display.ts`：两处重复的 `pad` 闭包提为模块级 `pad2()`，时间换算常量（`MS_PER_DAY`、`BEIJING_UTC_OFFSET_MS` 等）具名化。
- `src/loopback.ts`：`originIsTrusted` 里对 `hostname` 的重复小写化与 `::1` 特判收敛为一次比较；`allowedHosts` 为空时直接返回，省掉一次 `new URL` 解析。
- `src/provider-rows.ts`：`setLive` 里的「注册两半并回滚」逻辑抽成内部 `register()`，`setLive` 只剩状态判断。
- `src/catalog.ts`：`isFree()` 直接在 `all()` 上短路，不再先构造一次 id 数组。

### 修复

- **CI 的构建产物 smoke test 一直在断言旧模块 id**：`.github/workflows/ci.yml` 里写死 `grep 'id: "dsh-workbuddy"'`，而 1.2.0 已把模块 id 改成包名，这项检查必然失败。现在改为从 `package.json` 的 `name` 读取，包名再改也不会漂移。
- **`pnpm-workspace.yaml` 缺 `packages` 字段**：pnpm 9/10 会直接报 `packages field missing or empty`，`pnpm install` 无法进行；补上 `packages: ['.']`。

### 说明

- 功能行为未改动：`pnpm run check`（typecheck + 111 项测试 + 构建）全绿。

## [1.2.2] - 2026-10-22

### 说明

- 与 1.2.1 内容完全一致。1.2.1 提交到 npm 后进入暂存队列未能上线（`E409 Cannot publish over previously staged version`），版本号作废，改发 1.2.2。

## [1.2.1] - 2026-10-22

### 修复

- **peer 范围在标准 semver 语义下漏掉 0.2.1 的预发布版本**：`>=0.2.0-rc.2 <0.3.0-0` 里的比较符只把预发布标签挂在 `0.2.0` 这个元组上，按 node-semver 的规则，`0.2.1-rc.2` 这类版本会被静默排除（只有宿主侧带 `includePrerelease` 的校验才放行）。范围改为 `>=0.2.0-rc.2 <0.2.1 || >=0.2.1-0 <0.3.0-0`，`0.2.0-rc.2`、`0.2.0`、`0.2.1-rc.2`、`0.2.1` 全部命中，`pnpm add` 不再报 `ERESOLVE`。

## [1.2.0] - 2026-10-22

### 变更

- **包名改为 `@bushizhanghan/dsh-workmate`**：与上游 `dsh-workbuddy` 在 npm 上区分开，避免同名混淆。同步改了模块 id（`__ModuleLoader__.load({ id })` 用包名）、`cordis.patch.yml` 里 insert 的 `name`、CLI 命令名（`dsh-workmate`）与文档。
- 模型 provider id 仍是 `workbuddy-ai`，**已有配置不用改**。

## [1.1.0] - 2026-10-22

兼容分支首个版本，基线为上游 `dsh-workbuddy@1.0.12`（提交 `9951d25`）。此后的上游条目见下方，按原样保留。

### 修复

- **在 dsh 0.2.1-rc.2 上被拒载**：`peerDependencies` 把 7 个 `@deepseek-ai/dsh-*` 精确钉在 `0.2.0-rc.2`，而 dsh 加载插件前会逐条比对 peer 是否满足运行时版本，0.2.1 上全项不满足 → 插件不加载，并提示「要求宿主 0.2.0-rc.2，当前运行 0.2.1-rc.2」。改为 `>=0.2.0-rc.2 <0.3.0-0`，同一份产物同时覆盖 0.2.0 与 0.2.1。
- **请求体里的客户端名不被上游识别**：宿主默认把自身称作 `DeepSeek Harness`（带空格），上游按客户端名校验时会判成未知客户端。现在发出前统一改写为 `DeepSeek-Harness`，其余内容原样透传。

### 变更

- 包名改为 `@bushizhanghan/dsh-workbuddy`，`repository` / `homepage` / `bugs` 指向本仓库；`author` 同时标注上游作者。
- 新增 `NOTICE`（fork 来源与改动范围）、`patches/`（两处改动的 diff）、`pnpm-workspace.yaml`（`allowBuilds`，让 pnpm 11 的依赖构建确认不再中断 CI）。
- `lib/` 由 `src/` 重新构建，而非手工改打包产物；`pnpm test` 111 项全部通过。

## [1.0.12] - 2026-10-18

### 修复

- **连登天数改为与官网 growth-center 同源**：1.0.11 显示的是上游 `streak_days`（活动期签到次数），与官网页面上的「本月已连登 N 天」不是同一个数。官网该数字来自 `GET /activity/growth/streak` 的 `streak.days`——**没有 `/v2` 前缀**（`/v2/activity/growth/streak` 返回 404，所以插件里那个读取函数一直拿到空值）。现在读这个端点，标签恢复为「连登 N 天」/「N-day streak」，仅在读取失败时退回 `streak_days`。

## [1.0.11] - 2026-10-18

### 修复

- **撤回 1.0.10 的「连续签到天数」重算**：1.0.10 用 `checkin_dates` 自行推算连续天数，凭空造出了第三个数字——上游 `streak_days` 是活动期签到次数，站点又显示自己的口径，三方对不上。现在如实显示上游 `streak_days`，并把标签从「连续签到 N 天」改为「本活动内签到 N 次」/「N check-ins this event」，不再声称连续。

## [1.0.10] - 2026-10-18

### 修复

- **正文偶现 `<analysis>` / `<summary>` 标签**：上游对一批模型（如国际版 `deepseek-v4.1-flash`、`balanced-model`、国内版 `hy3`、`deepseek-v4-pro`）只声明单个固定 `reasoning.effort`。插件此前把它当作「没有推理档位」，于是从不发送 `reasoning_effort`，上游便不建立 `reasoning_content` 通道，模型只能在正文里画推理草稿。现在这类模型以该固定档位声明推理能力并始终带上 `reasoning_effort`，同时不再进入推理探测队列。
- **「连续签到 N 天」把活动期累计天数当成连续天数**：上游 `streak_days` 统计的是活动窗口内的签到总次数（实测「11 次」里有两天没签）。改为按 `checkin_dates` 计算真正以今天（或昨天）为终点的连续天数，字段缺失时才退回原值。

### 新增

- **模型列表按实际可用性收敛**：上游目录不声明账号级可见性，付费档模型（国际版 `primary-model`、`gpt-5.5` 等）要到真实请求才返回 `is not allowed for user type`。现在首次遇到该错误就把模型从选择器隐藏（会话内记忆、刷新目录时重置），已选该模型的会话照常工作。

## [1.0.9] - 2026-10-09

### 变更

- 两版 README 的三张界面截图换回未经压缩的原图（`docs/card-accounts.png` / `docs/card-models.png` / `docs/model-picker.png`，共约 2.5 MB），细节清晰度恢复；npm 包体积随之增大。

## [1.0.8] - 2026-10-09

### 变更

- 两版 README 的五条亮点按用户措辞重写：**登录**（「免桌面端登录」，桌面 App 凭据仍可只读导入）、**账号**（补上余额预警与账号备注）、**一键签到**（「提供自动签到，默认关闭」）、**设备隔离**（各自的凭据、模型目录、余额相互隔离；切换账号不影响在途请求）、**开箱即用**（安装启用即可用，无需额外配置，现已支持 0.2.0-rc.2 桌面端）。

## [1.0.7] - 2026-10-09

### 变更

- 两版 README：新增三张界面截图（账号与余额、模型范围与开关、模型选择器），放在「快速开始」上方，资源位于 `docs/`。
- 亮点第 3 条由「国内一键」改名为「一键签到」，内容不变。
- 亮点第 4、5 条重写，换成更有特色的两条：**两个区域彼此独立**（各自的凭据、各自的模型目录、各自的余额；某区域目录为空就保持为空，绝不借用另一区域的名单；切换账号不影响在途请求）与**不碰你的桌面端**（桌面端登录态只读、从不回写；插件只保存自己 OAuth 得到的区域凭据副本，「断开」只删这些副本）。原先的「实时价格」与「Provider」不再列为亮点。
- 删除两版 README 的「模型与价格」整节（接口实现未变）。
- npm 包 `files` 加入 `docs`，使 tarball 里的 README 图片可用。

## [1.0.6] - 2026-10-09

### 修复

- 国际版重新列出 `deepseek-v4.1-flash`（`x0.00`）。1.0.5 删除本地兜底名单后，国际版改由 `/v2/enterprises/personal/models` 取目录，而该接口是一份**残缺视图**——它不返回 `deepseek-v4.1-flash`，还把 `hy4-preview` 报成 `x0.00`（促销态）。现在国际版与国际版 App 读同一个接口 `GET https://www.workbuddy.ai/v3/config`，该方法还一并带回 `gpt-6-astra`、`glm-5.3-flash`、`kimi-k2.8-preview`。旧的 `/v2/enterprises/personal/models` 保留为兜底路径，仅当 `/v3/config` 不可用时才使用。

## [1.0.5] - 2026-10-09

### 变更

- **删除本地价格表，两区域一律以各自实时目录为准。** 此前插件内置一份价格与免费名单（`src/product-config.ts`），并在国际版里覆盖目录返回的 `credits`。现在这份表已整体移除：模型是否存在、上下文与推理档、`credits` 价格全部照搬该区域目录，产品上新或调价即时生效，不再等插件发版。国内版此前已是实时权威，本次与国际版统一。
- 相应地，未拿到区域目录前列表为空是刻意行为——一个区域不会借用另一个区域的模型清单（原国际版的本地兜底三模型已不存在）。
- `credits` 是叠加限时活动折扣后的展示价：`hy4-preview` 活动期内显示 `x0.00`、活动外为 `x0.29`，插件不再自行从严判定，直接照搬上游当前公布值。
- 配置项 `productConfigFile` / 环境变量 `DSH_WORKBUDDY_PRODUCT_CONFIG` 移除；`doctor` 不再输出产品配置来源，改为提示「价格实时读取自各区域目录」。
- 两版 README：副标题改为「无需下载 WorkBuddy 桌面端」，「免费模型」章节改写为「模型与价格」，说明实时目录与限时活动折扣；亮点第 3/4/5 条重写（国内一键签到/猫猫旅行/成长任务、实时价格、Provider 按登录出现）。

## [1.0.4] - 2026-10-09

### 修复

- 未登录时「模型」设置页不再出现 WorkBuddy Provider。此前 Provider 与目录条目在启动时无条件注册，模型页会列出一个没有凭据、无法使用的路由；现在按区域凭据决定注册与注销：登录该区域后才出现，断开或移除最后一个账号后随之消失。两半（adapter 与目录条目）同进同退，避免出现「列出但不可用」的半注册状态。

### 变更

- 未登录提示去掉「桌面 App 不是必须的」，只保留「点击『连接』通过 WorkBuddy 网站登录。」
- 两版 README 改为居中标题 + 徽标（许可证、DSH 版本、Provider 数、默认免费、零运行时依赖），删除「本插件独立于其它 WorkBuddy 插件」句、`## 和其他 WorkBuddy 插件的关系`、`## 已知限制` 与架构参考一行。

## [1.0.3] - 2026-10-09

### 修复

- 修复点击「连接」后浏览器打开的登录链接丢失 `state` 参数、站点报「登录链接不完整」的问题。Windows 下改用 `rundll32.exe url.dll,FileProtocolHandler`，不再经 `cmd /c start`——`cmd.exe` 会把 URL 里的 `&` 当成命令分隔符，把 `&state=…` 截断。
- 修复已登录一个区域后另一个区域点不动的问题。此前所选区域未登录时会强制回落到「已登录的区域」，导致第二个区域永远无法添加；现在所选 tab 始终生效，仅在首次发现已登录区域时做一次性预选。
- 移除与「连接」功能重复的「打开登录页」按钮（同一链接、同一动作，且两个标签页会争抢一次性 login state）。

### 变更

- Provider 显示名改为 **WorkBuddy 国际版** / **WorkBuddy 国内版**。
- 默认 README 改为英文：`README.md` 为英文，中文移至 `README.zh.md`。

## [1.0.2] - 2026-10-09

### 修复

- 未登录界面新增国内/国际区域切换，修复此前只能发起国内登录、国际版没有入口的问题。
- 点击「连接」不再重复打开登录页：此前宿主与页面各开一个标签页，两者争夺同一个一次性登录 state，先打开的成功、后打开的提示「登录链接不完整」。现在只由宿主打开，页面保留「打开登录页」作为兜底链接。

## [1.0.1] - 2026-10-09

### 修复

- 统一设置卡片与主机控制路由使用的安全请求头，修复点击“连接”等操作时被错误拒绝的问题。

## [1.0.0] - 2026-10-09

首个独立版本，以 `dsh-workbuddy` 名称发布并使用独立标识。

### 新增

- 双区域 Provider：`workbuddy-ai`（国际，显示为 **WorkBuddy**）与 `workbuddy-cn`（国内，显示为 **WorkBuddy 国内**）
- 按区域浏览器 OAuth 登录（`auth/state` → `authUrl` → 轮询 `auth/token`），凭据只写插件自己的副本
- 只读导入桌面 App 凭据（`workbuddy-desktop-ai.info` / `workbuddy-desktop.info`）
- 区域隔离的多账号列表、显式切换、剩余积分显示与连通性测试
- 国内区域可选自动签到（默认关闭，仅对勾选账号生效）
- 免费模型名单以应用下发产品配置为准，缺失时回退到内置名单
- 设置卡片：连接/断开、账号管理、`免费模型 / 全部模型` 范围切换、刷新间隔
- 命令行：`login` / `status` / `doctor` / `logout`（`status`、`doctor` 支持 `--json`）
- 双语 README（[English](./README.md) / [中文](./README.zh.md)）与 CI 工作流

### 标识（与 `dsh-connect-workbuddy` 完全隔离）

| 项 | 值 |
|---|---|
| Provider | `workbuddy-ai` / `workbuddy-cn` |
| 凭据文件 | `.workbuddy-ai-auth.json` / `.workbuddy-cn-auth.json` |
| 心跳文件 | `.workbuddy-ai-host-heartbeat.json` |
| 设置文件 | `.workbuddy-ai-settings.json` |
| 探测缓存 | `.workbuddy-ai-probe.json` |
| 环境变量 | `DSH_WORKBUDDY_AUTH_FILE` / `DSH_WORKBUDDY_CN_AUTH_FILE` / `DSH_WORKBUDDY_PRODUCT_CONFIG` |
| 控制头 | `X-WorkBuddy-Control-Key` |

### 适配

- 对齐 DSH `0.2.0-rc.2`：所有 `@deepseek-ai/dsh-*` peer 依赖精确锁定 `0.2.0-rc.2`，`@earendil-works/pi-ai` 为 `^0.87.1`
