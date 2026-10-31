# WB小灯 · 菜单栏 W 状态灯 开发文档

**一句话**：菜单栏上放一个加粗的 `W`，让你在干别的事的时候，一眼就能看出来 WorkBuddy 是在干活还是在闲着。

- 版本：**V2.6**（2026-10-28）—— ①V2.5 修「**没活儿了灯还一直绿闪不灭**」（快照 ✳ 判定无时效）；②**真修「换完提示音重启会丢」**——app 的 Bundle ID 与 NSUserDefaults suite 名同名，macOS 直接忽略该 suite，偏好只在进程内存里、**从不落盘**（详见 0.9）。两次修复都在 **.app 真实环境**做过对照验证。V2.5（2026-10-28）—— 修「**没活儿了灯还一直绿闪不灭**」：V2.2 的「快照有 ✳ 任务就不灭灯」**没有时效**，任务被压缩续跑/异常中断后 WorkBuddy 会把条目永远留在 state=working，灯就永远闪。V2.5 给它加上与 transcript 防挂死相同的 **15 分钟时效**（条目 lastActivityAt 超 15 分钟没动静就当它停了）。自测升 **65 项**（新增 5 项时效用例）。V2.4（2026-10-27）—— 「退出WB小灯」补上**方框叉图标**（xmark.square，13pt；重启项的循环箭头 V1.3 之前就有，退出这个源码里一直漏写，这次落盘，与 Codex V1.4 同步）。V2.3（2026-10-27）—— 面板删掉「最近动作」整行（刚哥定：没意义）。（2026-10-27 修两 bug）—— ①面板首行乱码：分块读块头切在汉字中间时旧码整块按 Latin-1 硬解 → 全行乱码；改为丢 1~3 个残字节重解 UTF-8（`utf8FromChunk`，三处统一）。②**快照里有 ✳ 任务就强制工作中、绝不灭灯**（transcript 判定在回合间隙会误报闲置）。同时：首行格式统一「**当前任务：任务名**」、字号与 Codex 统一 **15**。V2.1：今日调用行上方加分隔线。V2.0：面板改版。V1.9：今日消耗已用累计差值+10 分钟。V1.8：更换提示音。
- 状态：**已修正并运行中**（bundle id `io.wblight.menubar`，通用二进制 arm64 + x86_64）
- 自测：**60 项全部 PASS**（含 7.75 块头切汉字解码、7.5 消耗、7.7 提示音落盘闭环）+ 真实 transcript 全量重放验证
- 依赖：零第三方库。V1.3 起联网查积分余额（复用积分助手技能的登录态授权），忙闲判定依旧纯本机

---

## 0. V1.3 新增：积分余额挂 W 旁边（2026-10-25）

菜单栏显示从 `W` 变成 **`W 5508`** —— 绿色数字 = WorkBuddy「积分余额」，60 秒刷新一次，与忙闲红绿完全无关（W 灯语、闪烁、菜单全部维持原样）。

| 项 | 做法 |
|----|------|
| 数据源 | `POST https://www.codebuddy.cn/billing/meter/get-user-resource-summary`（无 `/v2` 前缀），返回 `data.TotalCredits` 字符串，与客户端「积分余额」一致（实测 5516.54，随消耗实时变小） |
| 凭据 | 复用积分助手技能的授权解析（`buddy_station._resolve_auth('auto')`），ObjC 不碰 token：NSTask 跑一段内嵌 python，只回传一个数字 |
| python | `/usr/bin/python3` 优先，托管版本（`~/.workbuddy/binaries/python/versions/*/bin/python`）兜底 |
| 显示 | 四舍五入取整（5516.54 → 5517），绿色 `(0.30,0.95,0.45)` 12pt Semibold，不闪烁 |
| 失败 | 保留上次真实值（宁旧勿假）；从未成功就只显示 `W`；数字没变不重绘 |
| 排障 | 环境变量 `WBLIGHT_DEBUG=1` 启动会打印 `[credits] val=… (raw=…)` |

**踩过的坑**：① 该接口 403 拒绝无 `User-Agent` 的请求（urllib 默认 UA 被网关拦），必须带 `Mozilla/5.0`；② 活动积分（加油站 `total_credits`=1800）和这个「积分余额」（5516）是两码事，别混。

---

## 0. V1.2 修了什么（重要）

V1.1 有一个**致命误判**，是刚哥实测发现的：**深度思考时灯还是红的，而且活干完了绿灯不灭。**

### 0.1 根因：`role` / `status` 字段的层级读错了

V1.1 用「工具调用审计日志的时间戳」当心跳。问题是**深度思考（reasoning）根本不调工具**——推理跑在云端，本地几乎零痕迹。于是：

- 思考 3 分钟 → 日志无新行 → 灯判「闲」→ **红灯**（错）
- 交付之后 → 兜底窗口还热着 → 灯还绿 → **绿灯不灭**（错）

V1.2 换成读**会话事件日志**（transcript）：`~/.workbuddy/projects/<项目>/<会话id>.jsonl`，逐事件追加，含 `reasoning`（深度思考原文）、`function_call`/`function_call_result`、`message`。

判定规则很简单：**看最后一行的「回合」走完没有**。不是「assistant 已交付完成」→ 还在干活 → 绿。

过程中踩了一个**大坑**，值得单独记：一开始写的是 `e["message"]["role"]`，实测取到全是空字符串，导致「已交付」被误判成「在途」，**绿灯永远不灭**。抓了 2017 行真实日志逐条验证才发现：**`role` 和 `status` 是顶层字段，不在 `message` 子字典里。**

```
✗ 错：{"type":"message", "message":{"role":"assistant","status":"completed"}}
✓ 对：{"type":"message", "role":"assistant", "status":"completed", "content":[...]}
```

### 0.2 顺手修掉的三个坑

| 坑 | 现象 | 修法 |
|----|------|------|
| `file-history-snapshot` / `ai-title` 插花落盘 | 这两类是**记账事件**（保存文件备份、生成会话标题），会在工具调用前后插进来。若恰好成为末行，会把"已交付"顶掉 → 假绿 | 新增 `isSubstantiveEvent`，判定时**从后往前跳过**记账事件，取最近的实质事件 |
| 用户按 ESC 打断 | 打断写入的是 `assistant + status=incomplete` + 内容 `Interrupted by user`。按"incomplete 即在途"会误判成绿，**一直绿到 15 分钟挂死保护** | 识别 `Interrupted by user` / `aborted` → 视为回合已结束 → 立刻红 |
| 兜底窗口拖出绿灯长尾 | 主判据之外再"并联"一个 20 秒心跳窗口，会让干完活后**多绿 20 秒** | 改成**真兜底**：只有读不到 transcript 时才启用心跳窗口 |

### 0.3 绿灯尾巴有多长

回合交付（`assistant completed`）落盘后，下一次巡检（**≤1 秒**）就判红。实测重放 269 次真实交付事件，**100% 正确转红，零假绿**。

---


## 0.1 「当前任务」一行（V1.4 新增 → V1.5 三修定稿，2026-10-25）

点开 W，「当前任务」显示**正在跑的任务名称**（如「一些小应用的开发」），就是 WorkBuddy 侧栏「任务」清单里的那个名字。

| 项 | 做法 |
|----|------|
| 数据源（首选） | 侧栏清单快照 `~/.workbuddy/<用户目录>/sidebar-list-snapshot.json`：取 `items[]` 里 `state=="working"`（侧栏的 ✳）且 `kind=="task"` 的条目的 `title` —— **与界面完全同源** |
| 多任务同时在跑 | 优先取与"正在被写入的 transcript 的会话 id"匹配的那条；否则取 `lastActivityAt` 最新的 |
| 兜底 | 没有 working 任务时，退回最新会话文件里的最后一个 `ai-title` |
| 走过的弯路 | ①复述用户最近一句话 —— 长会话里窗口够不到（15.9MB 文件、最近消息在 1.6MB 外）且会把压缩续跑指令当任务；②会话 ai-title —— 一次性会话标题滞后，不跟当前内容走。最终以侧栏快照为准 |
| 刷新 | 30 秒一次（独立于 1 秒忙闲巡检）；两边都取不到显示 `—` |
| 无关项 | 只展示，不参与忙闲红绿判定 |

## 0.2 V1.5 新增：面板显示「今日消耗 X 积分」（2026-10-25）

「今日调用」下面多一行**今日消耗**，告诉你今天烧了多少积分。

| 项 | 做法 |
|----|------|
| 为什么用差值法 | 官方 `get-user-resource-summary` 只有周期累计（CycleUsedCapacity），没有按天明细；已探测 11 个疑似明细端点，全部 404 |
| 基线 | 每天第一次拉到余额时记下当日基线，存 `~/Library/Application Support/WBLight/spent_day.txt`（格式 `YYYY-MM-DD 基线值`），跨重启持久化 |
| 计算 | 今日消耗 = 当日基线 − 当前余额；**10 分钟节流重算一次**（`kSpentRefresh=1800`，不重要指标别太频繁，2026-10-25 刚哥定）；余额变大（充值/补发）不倒扣，宁少勿假 |
| 颜色 | **数字红色**（与灯语红同色 1.0/0.30/0.28）；尾缀「（10 分钟更新）」**白字**（刚哥指定）；其余次要灰 |
| 格式 | <0.05 记 0；整数不带小数尾；否则留 1 位小数（纯函数 `formatSpent:`，自测覆盖） |
| 边界（说实话） | **当天中途第一次记录的话，之前已耗的不算**（基线只能从第一次拉到数起算）；灯开机自启后，基线在登录时即建立，整天基本准。数字没到 0.05 就显示 0，基线没就绪显示 — |

---

## 0.3 「更换提示音」子菜单（V1.8 新增，2026-10-26，整套移植自 Codex小灯）

- **位置**：「打开 WorkBuddy」下面、重启/退出上面。
- **功能与 Codex 完全一致**：
  - 「提示音：开启/关闭」开关，点一下切换，开启时立刻响一声验证；
  - 音源分三组扫描：`/System/Library/Sounds`（系统）、`/Library/Sounds`（其他）、`~/Library/Sounds`（自定义），后缀 aiff/aif/wav/caf/m4a/mp3，同名去重（系统音优先）——本机扫出 **16 个**；
  - 点某个音即选中并**当场试听**，前面打勾记住当前音；「试听当前提示音」单独兜底；
  - 偏好存 `NSUserDefaults` suite **`io.wblight.menubar`**（键 `SoundEnabled` / `SoundPath`），默认 **Tink.aiff**，重启后记忆保留；
- **触发点**：状态从忙（绿闪）转闲（红常亮）那一次 `refresh` 里响，即「活干完了叮一声」；防抖 1.2 秒（`kMinSoundGap`）。
- **实现注意（Codex 踩过的坑，原样带过来）**：`NSSound` 必须被 Delegate 持有，否则声音被提前释放（只响半声或不响）；`loadSoundPrefs` 必须在 `buildMenu` 之前调用，否则勾选态不对。

---

## 0.4 「今日消耗」为什么之前恒显示 0（V1.9 修复，2026-10-26）

**现象**：面板「今日消耗 0 积分」一动不动，30 分钟更新看着像失效。

**排查结论：接口是通的、30 分钟节流也在跑，错的是算法。**

- 后台 `get-user-resource-summary` 返回两套数：`TotalCredits`（余额）和各包的 `CycleUsedCapacity`（周期内**已用累计**）。
- V1.5 用「当日余额基线 − 当前余额」。而**每日签到会发 +100 积分**（后台确认：`today_checked_in=true / daily_credit=100`，已连续签到 19 天），
  发分把余额抬高，**抵消掉当天真实消耗** → 差值算出来是负数 → 被 `>0 才显示` 截成 0。
- 实测当天：基线 5258.99，当晚余额 5246.09 / 5259.57（两次采样，因为期间自己还在烧），**真实已耗 112.9 积分**，旧算法一直显示 0。

**修法**：改用「当日 `CycleUsedCapacity` 基线 − 当前」——
- `CycleUsedCapacity` 只增不减，签到/活动发分**不加**它，只对耦它，这才是真消耗；
- 包过期 / 周期翻转导致已用回落时的处理：记 0 并重置锚点，不显示负数；
- 基线文件格式从两段升级为三段：`日期 余额 已用累计`（旧两段文件自动兼容，首次取数时补锚点）。
- 去重同名的坑：`TotalCount: 21` 表示活动期间已发放 21 笔奖励，进一步说明"余额单调下降"这个假设根本不成立。

**同时**：刷新间隔 30 → **10 分钟**（刚哥改），面板尾缀文案同步为「（10 分钟更新）」。

---

## 0.5 面板改版：任务行第一行绿字加大（V2.0，2026-10-26 刚哥指定）

- **行序**：「当前任务」提到第一行（任务之下才是状态和动作，逻辑顺了）；「工作中·已忙」退到第二行。
- **任务行**：整行绿字（systemGreen），13 半粗（比其他行大一号）。
- **状态行**：与其他行同字号（11 常规）；文字白色（labelColor），**只有「已忙 3 分 12 秒」里的 3 和 12 两个数字绿色**；
  闲置/未运行时整行白字。忙闲灯语（菜单栏 W 的绿闪/红常亮）不变。
- 实测（探针逐字符采样）：`工@W 作@W 中@W … 0@G 分 0@G 秒@W` ✅
- **V2.1 补充**：「今日调用」行上方再加一条分隔线——上面三行是"现在状态"，下面两行是"今日统计"，隔开更清楚（刚哥指定）。

---

## 0.6 面板删掉「最近动作」整行（V2.3，2026-10-27 刚哥指定）

**一句话**：面板从三行变两行——「最近动作  xxxxx」这一行整个拿掉，只留「当前任务」+「工作中 · 已忙 X 分 X 秒」。

| 项 | 做法 |
|----|------|
| 删除范围 | 属性 `actionField`、创建它的 `makeField` 调用、`renderActionLine` 渲染块、面板行循环里的第三个元素 |
| 保留 | `lastAction` 解析函数与自测用例 3（解析链路不能断，以后想加回来随时能加） |
| 循环 | `for (NSTextField *f in @[self.taskField, self.statusField])` —— 两行 |

> 刚哥原话：「这个最近动作，这个也不要了，没有意义啊，删掉它，这一行」。
> 注意是**整行删**而不是留空：留空会在面板上留一道空白，高度不会自动收。

---

## 0.8 修「没活儿了灯还一直绿闪」（V2.5，2026-10-28 刚哥实测）

**一句话**：任务异常中断后，WorkBuddy 侧栏快照会**永远**留着一条 `state=working`；V2.2 的「有 ✳ 就不灭灯」没写时效，于是灯一直闪、不知道什么时候停。

| 项 | 内容 |
|----|------|
| 现象 | 没有任何任务在跑（侧栏静止、无进程），WB小灯 仍一直绿闪，几十分钟不停 |
| 根因 | V2.2 `snapshotHasWorkingTask` 只看 `state=="working"` 就强制忙——**没有时效**。任务被压缩续跑/异常中断后，WorkBuddy 不把条目翻回 idle，快照里就永远留着一条 working |
| 对照 | transcript「回合在途」信号**有** 15 分钟防挂死（`kMaxInFlight`），最多误绿 15 分钟会自愈；快照这条原本无自愈 |
| 修法 | 新增纯函数 `workingItemCounts:snapMtime:now:`：条目 `lastActivityAt`（毫秒）超 **15 分钟**（与 transcript 同参数）没动静就当它停了；`lastActivityAt` 缺失时退回快照文件 mtime 兜底 |
| 不变的纪律 | 「有任务不能灭灯」仍然成立——正在跑的任务 `lastActivityAt` 一直在刷新，永远不会被这条时效误伤；只有卡死的残留条目才会在 15 分钟后被放过 |
| 自测 | 4 节新增 5 项：1 分钟新鲜→忙 / 超 15 分钟→放过 / 无 lastActivityAt+快照新鲜→忙 / 无 lastActivityAt+快照旧→闲 / idle→闲。**65 项全部 PASS**（在打好包的 .app 里跑） |

**打包前的对照验证（真实数据，不是假的）**——用临时探针（`.app` 不打包它）同时保留「旧逻辑」和「新逻辑」，同一份真实快照 + 真实会话日志：

| 场景 | transcript 信号 | 快照·旧逻辑 V2.2 | 快照·新逻辑 V2.5+ | 总判定 |
|------|----------------|------------------|-------------------|--------|
| 此刻（确实在干活） | 忙（事件 2 秒前） | 忙 | 放过（条目 lastActivity 已 20 分钟没动） | **忙（灯闪）✅** |
| 静止后 8 分钟 | 忙（在途 8 分钟） | **忙（旧 bug：一直闪）** | 放过 | 忙 |
| 静止后 16 分钟 | 闲（超 15 分钟防挂死） | **忙（永远闪）** | 放过 | **闲（灯灭）✅** |
| 静止后 30 分钟 | 闲 | **忙（永远闪）** | 放过 | **闲（灯灭）✅** |

另用「把 working 条目的 lastActivityAt 往回拨」做四组定向对照：卡死 2 小时前→旧忙/新闲；静止 10 分钟→两者都忙（说明「有任务不灭灯」纪律没被破坏）；静止 16 分钟→两者分叉；30 秒前→两者都忙。

---

## 0.9 真修「换完提示音重启会丢」（V2.6，2026-10-28 打包前用 .app 实测查出）

**一句话**：V2.3 那次以为修好了，**其实没修到**。app 的 Bundle ID 就叫 `io.wblight.menubar`，代码却用同一个名字去 `initWithSuiteName:` → macOS 直接告警并忽略该 suite：

```
Using your own bundle identifier as an NSUserDefaults suite name
does not make sense and will not work.
```

返回的实例**不落盘，只活在进程内存里** → 菜单里换好提示音，重启 app 又回到默认「Tink」。

| 项 | 内容 |
|----|------|
| 为什么上次没发现 | V2.3 的自测跑的是**没打进 .app 的裸二进制**（没有 Info.plist、没有 Bundle ID），此时 suite 名不冲突、读写正常 → 自测一路 PASS。**测试环境和真实运行环境不一致，把自己骗了** |
| 修法 | `soundPrefs` 改用 `[NSUserDefaults standardUserDefaults]`——写的就是 `~/Library/Preferences/io.wblight.menubar.plist`，两个 app 的 Bundle ID 不同天然隔离。`--soundpref` 探针与自测同步改 |
| 同样的病 | Codex小灯 V1.5 同步修（Bundle ID `local.codex.light` 与 suite 同名） |
| 验证（决定性） | ① `defaults write io.wblight.menubar SoundEnabled -bool false / SoundPath /tmp/x.aiff` → 再让 **.app 里**的二进制去读：修前读到 `1\|`（空），修后读到 **`0\|/tmp/x.aiff`** ✅；② 自测 65 项在 **.app 内**跑，全部 PASS / 0 FAIL ✅ |
| 排查教训 | 偏好读写这类"看不见"的 bug，必须在**真实运行环境**（打包后的 app）里验，不能只信命令行裸二进制的自测 |

---

## 0.7 修两个「提示音偏好」的真问题（V2.3，2026-10-27 自测查出来的）

**一句话**：自测用例 7.7 一直显示 PASS，其实是**假 PASS**；顺藤摸出一个真 bug：换完提示音重启会丢。

**真 bug（影响使用）**：

| 项 | 内容 |
|----|------|
| 现象 | 菜单里换好提示音、开关提示音，重启 app 后又变回默认（「叮」） |
| 根因 | `soundPrefs` **每次调用都新建一个** `NSUserDefaults`（`initWithSuiteName:`），写完从不 `synchronize`；进程退出时改动还没送到 cfprefsd，就随进程没了 |
| 修法 | ① `soundPrefs` 改成 `dispatch_once` 进程内单例；② `saveSoundPrefs` 结尾加 `[d synchronize]` |

**假 PASS（影响可信度）**：

| 项 | 内容 |
|----|------|
| 假在哪 | 旧用例写完就直接拿一个新 Delegate 去读，没 `synchronize`；新实例读到的是**磁盘上的旧值**，本该 FAIL |
| 为什么一直 PASS | 上一轮用例把 `/tmp/x.aiff` 留在了磁盘上，下一轮读到的恰好就是这个值 —— **自己骗自己** |
| 顺带的破坏 | 还原时把「键本来不存在」写成 `SoundEnabled=false` / `SoundPath=""`，等于把默认「提示音开 + Tink」压成「关」 |
| 现在怎么测 | 写完 `synchronize` → **另起一个进程**（`WBLight --soundpref`，新增探针）把值读回来比对，等价于「重启 app 后再看还在不在」；还原时键原来不存在就 `removeObjectForKey`，绝不凭空造 false |
| 为什么不直接读 plist 文件 | cfprefsd 落盘是**异步**的，刚 `synchronize` 完文件里可能还没有，直接读文件会假失败（实测过：磁盘=(无)、实例=/tmp/x.aiff） |

> 已实测：连跑 3 次全 PASS；跑完 `defaults read io.wblight.menubar` 显示 domain 不存在（干净，没留脏数据）。

---

## 1. 灯语规则

| 顶条显示 | 含义 | 触发条件 |
|---------|------|---------|
| 绿色加粗 **W**，闪烁 | **忙** —— 正在执行任务 | WorkBuddy 在运行，且 60 秒内有过工具调用 |
| 红色加粗 **W**，常亮 | **闲** —— 空闲待命 | WorkBuddy 在运行，且当前没有进行中的回合 |
| 红色加粗 **W**，常亮 | **离线** —— 没开 | WorkBuddy 进程不存在 |

点一下这个 W，会展开一个小面板：

```
当前任务：一些小应用的开发       ← V2.2：格式带冒号；整行绿字 15 半粗（与 Codex 统一）
工作中 · 已忙 3 分 12 秒         ← V2.0：第二行，与其他行同字号；文字白色，只有 3/12 两个数字绿色；闲=「闲置中」白字
────────────────────────
今日调用  119 次  ·  活动于 3 秒前   ← V1.6：次数数字染绿（与灯语绿同色），其余保持次要灰
今日消耗  12.5 积分（10 分钟更新）  ← V1.9：已用累计差值法（不被签到发分干扰）；数字红色，尾缀白字
────────────────────────
打开 WorkBuddy
更换提示音 ▸                    ← V1.8 新增：子菜单（开关 + 16 个音源 + 试听当前提示音）
────────────────────────
重启 WB 小灯 🔄  ← V1.3 追加：与 Codex小灯 同款重启逻辑（等新实例拉起再退出 + open -n 兜底）；配「循环箭头」图标
退出WB小灯 ⊠    ← V1.7：改名；V2.4：补「方框叉」图标（源码此前一直缺失，这次落盘）
```

---

## 2. 判定原理：WorkBuddy 的「干活心跳」在哪

不猜、不轮询网络，直接读 WorkBuddy 在本机留下的活动痕迹：

| 信号 | 位置 | 更新时机 | 用来判断 |
|------|------|---------|---------|
| **在线** | 进程 `com.tencent.workbuddy.mac` | — | 在线（`NSRunningApplication`，O(1)） |
| **回合在途（主判据）** | `~/.workbuddy/projects/<项目>/<会话id>.jsonl` 末尾事件的 `type` / `role` / `status` | **每个事件**（含深度思考） | 忙 / 闲 |
| **干活心跳（兜底）** | `~/.workbuddy/audit-log/spool/*.jsonl` 行内 `timestamp` | 每次工具调用 | 仅当 transcript 读不到时启用 |
| **干活心跳（兜底）** | `~/.workbuddy/audit-log/YYYY-MM-DD.jsonl` 行内 `timestamp` | 当天归档 | 同上 |
| ~~最近动作~~（V2.3 起面板不再显示） | transcript 末条实质事件 / spool 末行 `commandPreview` | 同上 | 仅自测用例 3 保留，用于验证解析链路没断 |
| **当前任务**（V1.5 三修） | 侧栏快照 `sidebar-list-snapshot.json` 的 `state=working` 任务名；兜底最新会话 `ai-title` | 30 秒一次 | 面板显示"正在跑哪个任务" |
| **今日消耗**（V1.5） | 积分余额接口 + 本地基线 `~/Library/Application Support/WBLight/spent_day.txt` | 随余额 60 秒刷新 | 面板显示"今天烧了多少积分" |
| **更换提示音**（V1.8） | 系统音库扫描（/System、/Library、~/Library/Sounds；aiff/wav/caf/m4a/mp3）+ NSUserDefaults 偏好 | 点选即生效并试听 | 绿→红（忙完）时叮一声，防抖 1.2 秒 |
| 今日调用数 | spool + 当天归档的事件行数 | 同上 | 面板统计（带缓存） |

判定公式（源码里的纯函数 `isBusyOnline:heart:inFlight:inFlightSince:hasTranscript:now:window:`）：

```
有 transcript：
    忙 = 在线 && 末条实质事件不是「assistant 已交付」 && 该事件距今 < 15 分钟
无 transcript：
    忙 = 在线 && 最后一次工具调用距今 < 20 秒
```

**末条实质事件 → 判定对照表**：

| transcript 末行 | 判为 | 灯 |
|----------------|------|-----|
| `type=reasoning` | 在途（深度思考） | 绿 |
| `type=function_call` | 在途（工具执行中） | 绿 |
| `type=function_call_result` | 在途（想下一步） | 绿 |
| `type=message, role=user` | 在途（刚接单） | 绿 |
| `type=message, role=assistant, status=completed` | **已交付** | **红** |
| `type=message, role=assistant, status=incomplete`（普通） | 在途（正在回复） | 绿 |
| `type=message, role=assistant` + 内容含 `Interrupted by user` / `aborted` | 已被打断 | **红** |
| `type=file-history-snapshot` / `ai-title` | 记账事件，跳过，继续往前找 | — |

**为什么这个方案好**：

- **深度思考不再误红**：思考时 `reasoning` 事件是逐条落盘的，回合在途状态骗不了人
- **交付即转红**：`completed` 一落盘，下一次巡检（≤1 秒）就转红，没有绿灯长尾
- 不吃 CPU：只读文件**尾部 64KB**，不做全量扫描；每秒 stat 一次，开销可忽略
- 不需要配置 hook、不改 WorkBuddy 任何设置
- 离线自然就是红灯（没进程 → 没活动 → 红），语义天然对齐
- 不依赖联网，断网也准

**已知边界（说实话）**：

1. **挂死保护**：万一 agent 卡死、末条事件停在"在途"不动，最多绿 15 分钟（`kMaxInFlight`）后强制转红。
2. **等你确认时是红的**：判定的是"WorkBuddy 这一侧有没有在动"，它挂在等你点确认时确实没干活 —— 红灯是对的。
3. **换会话时以最新为准**：并行开多个会话时，取 mtime 最新的那个 transcript。哪个在动就显示哪个。

---

## 3. 完整源码（WBLight.m）

> 下面是 V2.2 的完整源码，共 1557 行，单文件、零依赖。
> 复制出去 `clang -fobjc-arc -framework Cocoa -o WBLight WBLight.m` 直接能编。

```objc
// WBLight.m — 菜单栏「W 灯」：一眼看出 WorkBuddy 是忙还是闲
//
//   忙（在线 + 正在干活）→ 绿色加粗 W，闪烁
//   闲（在线但没任务）    → 红色加粗 W，常亮
//   离线（WorkBuddy 没开）→ 红色加粗 W，常亮
//
// 判定依据（全部读本机文件与进程，不联网、不需要任何配置）。V1.2 双信号：
//   1) 在线：WorkBuddy 进程是否在运行（bundle id com.tencent.workbuddy.mac）
//   2) 忙 —— 只要 transcript 读得到，就以「回合在途」为唯一判据：
//      a) 主判据（在途 = 忙）——
//         ~/.workbuddy/projects/<项目>/<会话id>.jsonl 是会话事件日志，逐事件追加：
//           reasoning=深度思考  function_call/result=工具调用与结果
//           message(role=user)=用户消息  message(assistant,completed)=交付
//         看最新 transcript 的最后一行：只要不是"assistant 交付完成"，
//         说明回合还没走完（在思考/在调工具/在生成）→ 绿。
//         **末行变成"已交付"就立刻转红**（巡检 1 秒一次），不再有绿灯长尾。
//         这解决了 V1.1 的两个盲区：深度思考不写审计日志、不耗 CPU（推理在云端），
//         本地文件系统近乎无痕（V1.2 实测）；而"回合在途"状态骗不了人。
//         在途最长 15 分钟，防 agent 挂死导致永远绿。
//      b) 兜底（仅在读不到 transcript 时启用）——只认「工具调用审计日志」：
//           ~/.workbuddy/audit-log/spool/*.jsonl      （每次工具调用追加一行）
//           ~/.workbuddy/audit-log/YYYY-MM-DD.jsonl   （当天归档）
//         行内 timestamp（毫秒）距今 < 20 秒 → 绿。
//         注意：这是"兜底"不是"并联"，否则刚干完活的 20 秒内会一直被拖成绿灯。
//
//   ⚠️ 走过的弯路（V1.2 实测排除，别再试）：
//      · changes-index/*.json —— 用户打字/发消息/界面操作都会写，误报源，剔除
//      · sessions/*.json 的 lastHeartbeat —— 固定 30 秒一跳的"存活"心跳，剔除
//      · audit-log 目录自身 mtime —— 归档/轮转会动它，剔除
//      · 进程 CPU（proc_pid_rusage 实测全部 19 个 WorkBuddy 进程）—— 推理在云端、
//        本地流式渲染每秒 CPU 增量不足 0.01s，根本测不出"在生成"，弃用
//      · polling-lease-*.json（5 秒一跳的轮询租约）、claw-state/weixin/*（机器人
//        游标）—— 固定节奏写，与忙闲无关，剔除
//
// 编译：clang -fobjc-arc -framework Cocoa -o WBLight WBLight.m
// 自测：./WBLight --selftest

#import <Cocoa/Cocoa.h>

static NSString *const kWorkBuddyBundleID = @"com.tencent.workbuddy.mac";
static const NSTimeInterval kBusyWindow        = 20.0;   // 秒：心跳兜底窗口（仅当 transcript 读不到时生效）
static const NSTimeInterval kMaxInFlight       = 900.0;  // 秒：回合在途的最长时效（15分钟防挂死）
static const NSTimeInterval kPollInterval      = 1.0;    // 秒：状态巡检间隔
static const NSTimeInterval kSpentRefresh      = 600.0;  // 秒：今日消耗 10 分钟刷一次（2026-10-26 刚哥改为 10 分钟）
static const NSTimeInterval kMinSoundGap       = 1.2;    // 秒：完成提示音防抖间隔（避免连响）
static const NSTimeInterval kBlinkInterval     = 0.55;   // 秒：忙碌时的闪烁半周期
static const NSUInteger     kTailBytes         = 65536;  // 只读文件尾部这么多字节（避免全量读大日志）
static const NSUInteger     kTitleChunk        = 262144;  // 当前任务=ai-title：从后往前分块扫描，每块 256KB
static const NSUInteger     kTitleScanCap      = 8388608; // ai-title 最多往回扫 8MB（防超长会话失控）
static const NSTimeInterval kTaskRefresh       = 30.0;   // 当前任务 30 秒刷一次（独立于 1 秒忙闲巡检）

@interface Delegate : NSObject <NSApplicationDelegate, NSMenuDelegate>
@property (strong) NSStatusItem *item;
@property (strong) NSMenu *menu;
@property (strong) NSTimer *pollTimer, *blinkTimer;
@property (strong) NSTextField *statusField, *taskField, *countField, *spentField;   // V2.3：最近动作行删除（刚哥定：没意义）
@property (nonatomic) BOOL online, busy, blinkOn;
@property (nonatomic) NSTimeInterval lastActivity, busySince;
@property (copy)   NSString *lastAction;
@property (nonatomic) NSInteger todayCalls;
@property (nonatomic) NSInteger countCache;      // 调用次数缓存（按文件 size+mtime 失效）
@property (copy)       NSString *countCacheKey;
@property (nonatomic)  NSTimeInterval heartTime;      // 合并心跳（transcript 与审计日志取新）
@property (nonatomic)  BOOL       inFlight;           // 回合在途（最后一行不是交付）
@property (nonatomic)  NSTimeInterval inFlightSince;  // 在途起点（末行事件时刻）
@property (copy)       NSString *transcriptEvent;     // 末行事件中文名（菜单显示用）
@property (nonatomic)  BOOL       hasTranscript;         // 本次是否成功读到 transcript

// ---- Buddy 积分余额（挂 W 旁边，绿色）----
@property (nonatomic)  NSInteger  credits;              // 积分余额，-1 = 还没拿到
@property (nonatomic)  double    creditsRaw;            // 积分余额原始浮点（算今日消耗用）
@property (copy)       NSString  *currentTask;          // 最近一条用户消息 = 此刻在做的任务
@property (nonatomic)  BOOL       creditFetching;       // 防重入
@property (strong)     NSTimer   *creditTimer;
// ---- 今日消耗（V1.5）：当日基线差值法 ----
@property (copy)       NSString  *spentDayKey;          // 基线所属日期 YYYY-MM-DD
@property (nonatomic)  double     spentBaseline;        // 当日起点余额
@property (nonatomic)  double     usedBaseline;         // 当日起点「周期已用累计」（V1.9，-1 = 未知/旧格式文件）
@property (nonatomic)  BOOL       spentBaselineValid;   // 今天是否已有基线
@property (nonatomic)  double     todaySpent;           // 今日消耗 = 基线 − 当前（负值记 0）
@property (nonatomic)  NSTimeInterval lastSpentCalc;  // 上次重算今日消耗的时刻（10 分钟节流）
@property (nonatomic)  NSTimeInterval lastTaskFetch;  // 上次刷新当前任务的时刻（30 秒节流）

// ---- 完成提示音（V1.8，整套移植自 Codex小灯 同款：绿→红时「叮」一声）----
@property (nonatomic)  BOOL soundEnabled;                 // 开/关（NSUserDefaults 持久化）
@property (copy)       NSString *soundPath;               // 当前音源文件路径
@property (nonatomic)  NSTimeInterval lastSoundAt;        // 上次响的时刻（防抖）
@property (strong)     NSSound *player;                   // 必须持有，否则声音会被提前释放
@property (strong)     NSMenuItem *soundToggleItem;       // 「提示音：开启/关闭」
@property (strong)     NSMutableArray<NSMenuItem *> *soundChoiceItems; // 音源列表（打勾用）

// 积分查询
- (void)fetchCredits;
- (NSString *)firstAvailablePython;
- (NSString *)todayKey;
- (void)loadSpentBaseline;
- (void)updateSpentWithRaw:(double)raw used:(double)usedRaw;
+ (NSString *)formatSpent:(double)v;   // 纯函数，便于自测
- (void)restartApp:(id)sender;
- (NSString *)taskTitleFromSidebarSnapshot:(NSString *)path sessionId:(NSString *)sid;  // 侧栏快照里 state=working 的任务名
- (NSString *)latestTitleInFile:(NSString *)path;             // 会话文件里最后一个 ai-title（兜底）
- (NSString *)utf8FromChunk:(NSData *)d;                // 分块读的 UTF-8 兜底（V2.2：不再用 Latin-1）
- (BOOL)snapshotHasWorkingTask;                          // 快照里有 ✳ 任务 → 强制工作中（V2.2 不灭灯）

// 提示音（V1.8，与 Codex小灯 同款）
- (NSUserDefaults *)soundPrefs;
- (void)loadSoundPrefs;
- (void)saveSoundPrefs;
- (NSString *)defaultSoundPath;
- (NSArray<NSDictionary<NSString *, NSString *> *> *)availableSounds;
- (void)refreshSoundMenuItems;
- (void)playCompletionSound;
- (void)toggleSound:(id)sender;
- (void)selectSound:(id)sender;
- (void)previewSound:(id)sender;

// 忙闲判定（纯函数，便于自测）
- (BOOL)isBusyOnline:(BOOL)online heart:(NSTimeInterval)heart
            inFlight:(BOOL)inFlight inFlightSince:(NSTimeInterval)since
       hasTranscript:(BOOL)hasTranscript
                 now:(NSTimeInterval)now window:(NSTimeInterval)w;
- (NSString *)strField:(NSDictionary *)e key:(NSString *)k;
- (BOOL)isSubstantiveEvent:(NSDictionary *)e;
- (BOOL)isInterruptedMessage:(NSDictionary *)e;
- (BOOL)eventInFlight:(NSDictionary *)e;
- (NSString *)eventName:(NSDictionary *)e;
- (NSArray<NSDictionary *> *)tailEventsOf:(NSString *)path bytes:(NSUInteger)bytes limit:(NSInteger)limit;
- (void)refreshCurrentTask;
- (int)replayTranscript;
- (NSString *)stateLine;
- (int)runSelfTest;
@end

@implementation Delegate

// ================= 探测：进程 / 心跳 / 最近动作 =================

- (NSString *)wbHome {
    return [NSHomeDirectory() stringByAppendingPathComponent:@".workbuddy"];
}

// WorkBuddy 是否在运行。
// 主路径按 bundle id；万一对方装的是 id 不同的变体（分发渠道不同会有这种情况），
// 再按运行中进程的名字兜底找一遍 —— 否则对方那边会「永远是红灯」，看着像坏了。
- (BOOL)workBuddyRunning {
    NSArray<NSRunningApplication *> *apps =
        [NSRunningApplication runningApplicationsWithBundleIdentifier:kWorkBuddyBundleID];
    if (apps.count > 0) return YES;
    for (NSRunningApplication *a in [[NSWorkspace sharedWorkspace] runningApplications]) {
        NSString *n = a.localizedName ?: @"";
        if ([n rangeOfString:@"WorkBuddy" options:NSCaseInsensitiveSearch].location != NSNotFound)
            return YES;
    }
    return NO;
}

// 定位 WorkBuddy 的 app 位置（菜单里「打开 WorkBuddy」用）：bundle id 优先，名字兜底
- (NSURL *)workBuddyAppURL {
    NSURL *u = [[NSWorkspace sharedWorkspace] URLForApplicationWithBundleIdentifier:kWorkBuddyBundleID];
    if (u) return u;
    for (NSRunningApplication *a in [[NSWorkspace sharedWorkspace] runningApplications]) {
        NSString *n = a.localizedName ?: @"";
        if ([n rangeOfString:@"WorkBuddy" options:NSCaseInsensitiveSearch].location != NSNotFound)
            return a.bundleURL;
    }
    return nil;
}

// 目录下最新文件的修改时间（Unix 秒）；目录不存在或无文件返回 0 —— 仅作兜底
- (NSTimeInterval)latestMTimeIn:(NSString *)dir {
    NSFileManager *fm = [NSFileManager defaultManager];
    NSArray<NSString *> *names = [fm contentsOfDirectoryAtPath:dir error:nil];
    NSTimeInterval best = 0;
    for (NSString *n in names) {
        if ([n hasPrefix:@"."]) continue;
        NSDictionary *attr = [fm attributesOfItemAtPath:[dir stringByAppendingPathComponent:n] error:nil];
        NSDate *d = attr[NSFileModificationDate];
        if (d && d.timeIntervalSince1970 > best) best = d.timeIntervalSince1970;
    }
    return best;
}

- (NSString *)spoolDir {
    return [[self wbHome] stringByAppendingPathComponent:@"audit-log/spool"];
}

// 只读文件尾部 kTailBytes 字节，从后往前返回第一条能解析成 JSON 字典的行
// （避免每秒全量读 100KB+ 的大日志；末行若是半截写入，会自动往前找）
- (NSDictionary *)tailJSONOf:(NSString *)path {
    NSFileHandle *fh = [NSFileHandle fileHandleForReadingAtPath:path];
    if (!fh) return nil;
    unsigned long long size = [fh seekToEndOfFile];
    if (size == 0) { [fh closeFile]; return nil; }
    unsigned long long back = MIN((unsigned long long)kTailBytes, size);
    [fh seekToFileOffset:size - back];
    NSData *d = [fh readDataToEndOfFile];
    [fh closeFile];
    if (!d.length) return nil;
    NSString *text = [self utf8FromChunk:d];   // V2.2：块头残字节兜底，不再按 Latin-1 硬解
    if (!text.length) return nil;
    NSArray<NSString *> *lines = [text componentsSeparatedByString:@"\n"];
    for (NSInteger i = (NSInteger)lines.count - 1; i >= 0; i--) {
        NSString *line = [lines[(NSUInteger)i] stringByTrimmingCharactersInSet:
                              [NSCharacterSet whitespaceAndNewlineCharacterSet]];
        if (line.length < 2) continue;
        NSDictionary *j = [NSJSONSerialization JSONObjectWithData:[line dataUsingEncoding:NSUTF8StringEncoding]
                                                          options:0 error:nil];
        if ([j isKindOfClass:[NSDictionary class]] && j.count) return j;
    }
    return nil;
}

// spool 目录下 mtime 最新的 jsonl 文件路径
- (NSString *)newestSpoolFile {
    NSFileManager *fm = [NSFileManager defaultManager];
    NSString *dir = [self spoolDir];
    NSString *best = nil; NSTimeInterval newest = 0;
    for (NSString *n in [fm contentsOfDirectoryAtPath:dir error:nil]) {
        if (![n hasSuffix:@".jsonl"]) continue;
        NSString *p = [dir stringByAppendingPathComponent:n];
        NSDictionary *attr = [fm attributesOfItemAtPath:p error:nil];
        NSTimeInterval t = [attr[NSFileModificationDate] timeIntervalSince1970];
        if (t > newest) { newest = t; best = p; }
    }
    return best;
}

// 当天归档文件：audit-log/YYYY-MM-DD.jsonl
- (NSString *)todayArchiveFile {
    NSDateFormatter *df = [[NSDateFormatter alloc] init];
    df.dateFormat = @"yyyy-MM-dd";
    df.timeZone = [NSTimeZone localTimeZone];
    return [[self wbHome] stringByAppendingPathComponent:
            [NSString stringWithFormat:@"audit-log/%@.jsonl", [df stringFromDate:[NSDate date]]]];
}

// 日志行里的 timestamp（毫秒）→ Unix 秒；取不到返回 0
- (NSTimeInterval)secondsFromEvent:(NSDictionary *)j {
    id ts = j[@"timestamp"];
    if (![ts isKindOfClass:[NSNumber class]]) return 0;
    double v = [ts doubleValue];
    if (v <= 0) return 0;
    return v > 1e12 ? v / 1000.0 : v;      // 毫秒时间戳
}

// 干活心跳：只认「工具调用日志」—— spool 缓冲区 + 当天归档，取最新一条动作的时刻
// 刻意不读 changes-index / sessions / audit-log 目录 mtime（见文件头三个陷阱）
- (NSTimeInterval)lastActivityTime {
    NSFileManager *fm = [NSFileManager defaultManager];
    NSMutableArray<NSString *> *files = [NSMutableArray array];
    NSString *dir = [self spoolDir];
    for (NSString *n in [fm contentsOfDirectoryAtPath:dir error:nil])
        if ([n hasSuffix:@".jsonl"]) [files addObject:[dir stringByAppendingPathComponent:n]];
    NSString *arch = [self todayArchiveFile];
    if ([fm fileExistsAtPath:arch]) [files addObject:arch];

    NSTimeInterval best = 0;
    for (NSString *p in files) {
        NSTimeInterval t = [self secondsFromEvent:[self tailJSONOf:p]];
        if (t > best) best = t;
    }
    if (best > 0) return best;
    return [self latestMTimeIn:dir];    // 兜底：日志在但 timestamp 解析不出时用 mtime
}

// 今天的工具调用条数：spool 缓冲 + 当天归档，按文件 size+mtime 做缓存（避免每秒重扫大文件）
- (NSInteger)todayCallCount {
    NSFileManager *fm = [NSFileManager defaultManager];
    NSMutableArray<NSString *> *files = [NSMutableArray array];
    NSString *dir = [self spoolDir];
    for (NSString *n in [fm contentsOfDirectoryAtPath:dir error:nil])
        if ([n hasSuffix:@".jsonl"]) [files addObject:[dir stringByAppendingPathComponent:n]];
    NSString *arch = [self todayArchiveFile];
    if ([fm fileExistsAtPath:arch]) [files addObject:arch];
    if (files.count == 0) return 0;

    NSMutableString *key = [NSMutableString string];
    for (NSString *p in [files sortedArrayUsingSelector:@selector(compare:)]) {
        NSDictionary *a = [fm attributesOfItemAtPath:p error:nil];
        [key appendFormat:@"%@:%llu:%.0f|", p,
             [a[NSFileSize] unsignedLongLongValue],
             [a[NSFileModificationDate] timeIntervalSince1970]];
    }
    if (self.countCacheKey && [self.countCacheKey isEqualToString:key]) return self.countCache;

    NSInteger n = 0;
    for (NSString *p in files) {
        NSString *text = [NSString stringWithContentsOfFile:p encoding:NSUTF8StringEncoding error:nil];
        if (!text.length) continue;
        for (NSString *line in [text componentsSeparatedByString:@"\n"])
            if ([line containsString:@"\"eventType\""]) n++;
    }
    self.countCacheKey = key;
    self.countCache = n;
    return n;
}

// 最近一条动作：读 spool 最新文件的尾部，取 commandPreview（没有则取事件名）
- (NSString *)readLastAction {
    NSString *p = [self newestSpoolFile];
    if (!p) return @"—";
    NSDictionary *j = [self tailJSONOf:p];
    if (!j) return @"—";
    NSString *cmd = j[@"commandPreview"];
    if ([cmd isKindOfClass:[NSString class]] && cmd.length) return [self oneLine:cmd];
    NSString *ev = j[@"eventType"];
    if ([ev isKindOfClass:[NSString class]] && ev.length) return ev;
    return @"—";
}

// 把多行命令压成一行并截断
- (NSString *)oneLine:(NSString *)s {
    NSString *t = [[s componentsSeparatedByCharactersInSet:[NSCharacterSet newlineCharacterSet]]
                       componentsJoinedByString:@" "];
    while ([t containsString:@"  "]) t = [t stringByReplacingOccurrencesOfString:@"  " withString:@" "];
    return [self truncate:t max:58];
}

- (NSString *)truncate:(NSString *)s max:(NSUInteger)max {
    if (s.length <= max) return s;
    return [[s substringToIndex:max] stringByAppendingString:@"…"];
}

// ================= transcript 回合状态（V1.2 主信号） =================

// 最新的会话 transcript：~/.workbuddy/projects/*/*.jsonl 里 mtime 最新的那个
- (NSString *)latestTranscript {
    NSString *dir = [[self wbHome] stringByAppendingPathComponent:@"projects"];
    NSFileManager *fm = [NSFileManager defaultManager];
    NSArray<NSString *> *projects = [fm contentsOfDirectoryAtPath:dir error:nil];
    NSString *best = nil; NSTimeInterval newest = 0;
    NSDateFormatter *df = nil;
    for (NSString *proj in projects) {
        NSString *pdir = [dir stringByAppendingPathComponent:proj];
        for (NSString *n in [fm contentsOfDirectoryAtPath:pdir error:nil]) {
            if (![n hasSuffix:@".jsonl"]) continue;
            NSString *p = [pdir stringByAppendingPathComponent:n];
            NSDictionary *a = [fm attributesOfItemAtPath:p error:nil];
            NSTimeInterval t = [a[NSFileModificationDate] timeIntervalSince1970];
            if (t > newest) { newest = t; best = p; }
        }
    }
    return best;
}

// assistant 中间态是否为"打断/中止"（用户按 ESC、编译 aborted 等）——这类意味着回合已结束
- (BOOL)isInterruptedMessage:(NSDictionary *)e {
    id c = e[@"content"];
    if (![c isKindOfClass:[NSArray class]]) return NO;
    for (id b in (NSArray *)c) {
        if (![b isKindOfClass:[NSDictionary class]]) continue;
        NSString *txt = [self strField:b key:@"text"];
        if ([txt containsString:@"Interrupted by user"] || [txt hasPrefix:@"aborted"]
            || [txt containsString:@"Request interrupted"]) return YES;
    }
    return NO;
}

// 事件行 → 中文动作名（菜单显示）
// ⚠️ 实测（V1.2 把真实 transcript 上千行逐条 dump 验证过）：role / status 是「顶层字段」，
//    不在 message 子字典里！写成 e[@"message"][@"role"] 会永远取到空 → 交付误判为在途。
- (NSString *)eventName:(NSDictionary *)e {
    NSString *type   = [self strField:e key:@"type"];
    NSString *role   = [self strField:e key:@"role"];
    if ([type isEqualToString:@"reasoning"])            return @"深度思考中";
    if ([type isEqualToString:@"function_call"])        return @"调用工具";
    if ([type isEqualToString:@"function_call_result"]) return @"工具已返回 · 想下一步";
    if ([type isEqualToString:@"message"]) {
        if ([role isEqualToString:@"user"])      return @"刚接到新任务";
        if ([role isEqualToString:@"assistant"]) {
            if ([[self strField:e key:@"status"] isEqualToString:@"completed"]) return @"已交付回复";
            if ([self isInterruptedMessage:e])  return @"已被打断";
            return @"正在回复";
        }
        return @"消息";
    }
    if ([type isEqualToString:@"ai-title"])             return @"整理会话标题";
    if ([type isEqualToString:@"file-history-snapshot"])return @"记录文件快照";
    return type.length ? type : @"有新事件";
}

- (NSString *)strField:(NSDictionary *)e key:(NSString *)k {
    id v = e[k];
    return [v isKindOfClass:[NSString class]] ? v : @"";
}

// 是否"实质事件"：能反映回合进度的。记账类（file-history-snapshot / ai-title）
// 会在工具调用前后插花落盘，若恰好成为末行会把"已交付"顶掉、造成假绿，故跳过。
- (BOOL)isSubstantiveEvent:(NSDictionary *)e {
    NSString *t = [self strField:e key:@"type"];
    return [t isEqualToString:@"reasoning"]
        || [t isEqualToString:@"function_call"]
        || [t isEqualToString:@"function_call_result"]
        || [t isEqualToString:@"message"];
}

// 末行事件是否"回合在途"（不是 assistant 最终交付就是还在干活）
- (BOOL)eventInFlight:(NSDictionary *)e {
    NSString *type   = [self strField:e key:@"type"];
    NSString *role   = [self strField:e key:@"role"];
    if ([type isEqualToString:@"message"]) {
        if ([role isEqualToString:@"user"]) return YES;                     // 刚接单，还没回
        if ([role isEqualToString:@"assistant"]) {
            if ([[self strField:e key:@"status"] isEqualToString:@"completed"]) return NO;  // 已交付
            if ([self isInterruptedMessage:e]) return NO;                  // 被打断/中止 → 回合已结束
            return YES;                                                    // incomplete = 在途
        }
        return YES;
    }
    if (type.length == 0) return NO;                                        // 解析不出 → 不算
    return YES;                                                             // reasoning / function_call / result = 在途
}

// 读文件尾部，从后往前返回最多 limit 条能解析成字典的行（新的在前）；bytes 控制回读字节数
- (NSArray<NSDictionary *> *)tailEventsOf:(NSString *)path bytes:(NSUInteger)bytes limit:(NSInteger)limit {
    NSFileHandle *fh = [NSFileHandle fileHandleForReadingAtPath:path];
    if (!fh) return @[];
    unsigned long long size = [fh seekToEndOfFile];
    if (size == 0) { [fh closeFile]; return @[]; }
    unsigned long long back = MIN((unsigned long long)bytes, size);
    [fh seekToFileOffset:size - back];
    NSData *d = [fh readDataToEndOfFile];
    [fh closeFile];
    if (!d.length) return @[];
    NSString *text = [self utf8FromChunk:d];   // V2.2：块头残字节兜底，不再按 Latin-1 硬解
    if (!text.length) return @[];
    NSArray<NSString *> *lines = [text componentsSeparatedByString:@"\n"];
    NSMutableArray<NSDictionary *> *out = [NSMutableArray array];
    for (NSInteger i = (NSInteger)lines.count - 1; i >= 0 && (NSInteger)out.count < limit; i--) {
        NSString *line = [lines[(NSUInteger)i] stringByTrimmingCharactersInSet:
                              [NSCharacterSet whitespaceAndNewlineCharacterSet]];
        if (line.length < 2) continue;
        NSDictionary *j = [NSJSONSerialization JSONObjectWithData:[line dataUsingEncoding:NSUTF8StringEncoding]
                                                          options:0 error:nil];
        if ([j isKindOfClass:[NSDictionary class]] && j.count) [out addObject:j];
    }
    return out;
}

// 读最新 transcript 末行"实质事件" → @{time, inFlight, name}；读不到返回 nil
- (NSDictionary *)transcriptState {
    NSString *p = [self latestTranscript];
    if (!p) return nil;
    NSArray<NSDictionary *> *evs = [self tailEventsOf:p bytes:kTailBytes limit:120];   // 新的在前（忙闲判定用小窗口）
    NSDictionary *e = nil;
    for (NSDictionary *cand in evs)
        if ([self isSubstantiveEvent:cand]) { e = cand; break; }
    if (!e) e = evs.firstObject;
    if (!e) return nil;
    NSTimeInterval ts = [self secondsFromEvent:e];
    if (ts <= 0) {
        NSDictionary *attr = [[NSFileManager defaultManager] attributesOfItemAtPath:p error:nil];
        NSDate *d = attr[NSFileModificationDate];
        ts = d ? d.timeIntervalSince1970 : 0;
    }
    return @{@"time": @(ts),
             @"inFlight": @([self eventInFlight:e]),
             @"name": [self eventName:e]};
}

// 当前任务 = 最新会话的 ai-title（WorkBuddy 侧栏「任务」清单里显示的那个名称，V1.5 三修）——
// 刚哥指正：要的是任务名称，不是复述他的原话（复述那版会把压缩续跑指令当任务）。
// 标题存在会话自己的 transcript 里的 ai-title 事件（aiTitle 字段），从后往前分块扫描
// 取最后一个（标题可能中途改过）；预筛含 "aiTitle" 的行，绝大多数行不进 JSON 解析；
// 块边界向老数据重叠 1KB，防行被切断漏检；最多回扫 8MB 兜底。
// ---- UTF-8 分块读的统一兜底（V2.2 修乱码）----
// 按字节读尾/分块时，块起点可能落在多字节字符中间，整块 initWithData:encoding:UTF8 会返回 nil。
// 旧写法此时退化 Latin-1 硬解 —— Latin-1 对任何字节都"解得出"，于是整块中文变成
// 'æ æå¨ç¼' 这种乱码（2026-10-26 刚哥发现）。正解：丢掉块头不完整的 1~3 个残字节再解 UTF-8；
// 实在解不开就返回 nil，让调用方跳过这一块，绝不按 Latin-1 硬解。
- (NSString *)utf8FromChunk:(NSData *)d {
    NSString *t = [[NSString alloc] initWithData:d encoding:NSUTF8StringEncoding];
    for (NSUInteger trim = 1; !t && trim <= 3 && d.length > trim; trim++)
        t = [[NSString alloc] initWithData:[d subdataWithRange:NSMakeRange(trim, d.length - trim)]
                                  encoding:NSUTF8StringEncoding];
    return t;
}

- (NSString *)latestTitleInFile:(NSString *)path {
    NSFileHandle *fh = [NSFileHandle fileHandleForReadingAtPath:path];
    if (!fh) return nil;
    unsigned long long size = [fh seekToEndOfFile];
    unsigned long long scanned = 0, offset = size;
    NSString *found = nil;
    while (offset > 0 && scanned < kTitleScanCap) {
        unsigned long long back = MIN((unsigned long long)kTitleChunk, offset);
        unsigned long long readFrom = offset - back;
        unsigned long long overlap = (offset < size) ? MIN((unsigned long long)1024, readFrom) : 0;
        readFrom -= overlap; back += overlap;       // 向老数据重叠 1KB，防行被块边界切断
        scanned += back;
        offset = readFrom;
        [fh seekToFileOffset:readFrom];
        NSData *d = [fh readDataOfLength:(NSUInteger)back];
        NSString *text = [self utf8FromChunk:d];   // V2.2：块头残字节兜底，不再按 Latin-1 硬解
        if (!text.length) continue;
        NSArray<NSString *> *lines = [text componentsSeparatedByString:@"\n"];
        for (NSInteger i = (NSInteger)lines.count - 1; i >= 0; i--) {
            NSString *line = lines[(NSUInteger)i];
            if (line.length < 30 || ![line containsString:@"\"aiTitle\""]) continue;   // 廉价预筛
            NSDictionary *j = [NSJSONSerialization JSONObjectWithData:[line dataUsingEncoding:NSUTF8StringEncoding]
                                                               options:0 error:nil];
            if (![j isKindOfClass:[NSDictionary class]]) continue;
            id t = j[@"aiTitle"];
            if ([t isKindOfClass:[NSString class]] && [(NSString *)t length] > 0) { found = t; break; }
        }
        if (found) break;
    }
    [fh closeFile];
    if (!found) return nil;
    if (found.length > 30) found = [[found substringToIndex:30] stringByAppendingString:@"…"];
    return found;
}

// 当前任务首选：侧栏清单快照（与 WorkBuddy 界面同源）——
// ~/.workbuddy/<用户目录>/sidebar-list-snapshot.json 的 items[] 里，
// state=="working"（侧栏的 ✳）且 kind=="task" 的条目就是"正在做的任务"；
// 多个同时 working 时取 lastActivityAt 最新的。没有 working → 返回 nil（上层退回 ai-title）。
- (NSString *)taskTitleFromSidebarSnapshot:(NSString *)path sessionId:(NSString *)sid {
    NSData *d = [NSData dataWithContentsOfFile:path];
    if (!d.length) return nil;
    NSDictionary *root = [NSJSONSerialization JSONObjectWithData:d options:0 error:nil];
    if (![root isKindOfClass:[NSDictionary class]]) return nil;
    NSArray *items = root[@"items"];
    if (![items isKindOfClass:[NSArray class]]) return nil;
    NSString *best = nil, *idBest = nil;
    double bestT = 0;
    for (NSDictionary *it in items) {
        if (![it isKindOfClass:[NSDictionary class]]) continue;
        if (![it[@"state"] isEqualToString:@"working"]) continue;
        if ([it[@"kind"] isKindOfClass:[NSString class]] && ![it[@"kind"] isEqualToString:@"task"]) continue;
        id t = it[@"title"];
        if (![t isKindOfClass:[NSString class]] || [(NSString *)t length] == 0) continue;
        double act = [it[@"lastActivityAt"] doubleValue];   // 毫秒
        BOOL idMatch = (sid.length && [it[@"id"] isKindOfClass:[NSString class]] &&
                        [(NSString *)it[@"id"] isEqualToString:sid]);
        if (idMatch) { idBest = t; break; }              // 正在被写 transcript 的会话优先
        if (best == nil || act >= bestT) { bestT = act; best = t; }
    }
    if (idBest) return idBest;   // 活跃会话精确匹配最优先
    if (!best) return nil;
    if (best.length > 30) best = [[best substringToIndex:30] stringByAppendingString:@"…"];
    return best;
}

// 侧栏快照：~/.workbuddy/*/sidebar-list-snapshot.json 取 mtime 最新的（用户目录名不写死）
// 侧栏快照里有没有「正在跑」的任务（state==working 且 kind==task）——
// 有就强制工作中，绝不灭灯（V2.2，2026-10-27 刚哥定：有任务就不能灭灯）。
// transcript 判定在回合间隙/压缩续跑时会误报闲置，快照的 ✳ 才是 WorkBuddy 界面同源的事实。
// 纯函数：单条 working 条目还算不算「正在跑」（V2.5 加时效，便于自测）
// V2.5 起因（2026-10-28 刚哥实测）：任务被压缩续跑/异常中断后，WorkBuddy 会把条目永远留在
// state=working，V2.2 的"有 ✳ 就不灭灯"没有时效 → 没活儿了灯还一直绿闪，不知道什么时候停。
// 修法：条目 lastActivityAt（毫秒）超过 kMaxInFlight（15 分钟，与 transcript 防挂死同参数）
// 没动静就当它停了；lastActivityAt 缺失时退回快照文件 mtime 兜底。
- (BOOL)workingItemCounts:(NSDictionary *)it snapMtime:(NSTimeInterval)snapM now:(NSTimeInterval)now {
    if (![it isKindOfClass:[NSDictionary class]]) return NO;
    if (![it[@"state"] isEqualToString:@"working"]) return NO;
    if ([it[@"kind"] isKindOfClass:[NSString class]] && ![it[@"kind"] isEqualToString:@"task"]) return NO;
    NSTimeInterval act = [it[@"lastActivityAt"] doubleValue] / 1000.0;   // 毫秒 → 秒
    if (act <= 0) act = snapM;
    return act > 0 && (now - act) < kMaxInFlight;
}

- (BOOL)snapshotHasWorkingTask {
    NSString *p = [self latestSidebarSnapshot];
    if (!p) return NO;
    NSData *d = [NSData dataWithContentsOfFile:p];
    if (!d.length) return NO;
    NSDictionary *root = [NSJSONSerialization JSONObjectWithData:d options:0 error:nil];
    if (![root isKindOfClass:[NSDictionary class]]) return NO;
    NSTimeInterval now = [[NSDate date] timeIntervalSince1970];
    NSDate *snapM = [[NSFileManager defaultManager] attributesOfItemAtPath:p error:nil][NSFileModificationDate];
    NSTimeInterval snapMT = snapM ? snapM.timeIntervalSince1970 : 0;
    for (NSDictionary *it in root[@"items"])
        if ([self workingItemCounts:it snapMtime:snapMT now:now]) return YES;
    return NO;
}

- (NSString *)latestSidebarSnapshot {
    NSString *dir = [NSHomeDirectory() stringByAppendingPathComponent:@".workbuddy"];
    NSArray *entries = [[NSFileManager defaultManager] contentsOfDirectoryAtPath:dir error:nil];
    NSString *best = nil;
    NSDate *bestDate = nil;
    for (NSString *e in entries) {
        NSString *p = [dir stringByAppendingPathComponent:[NSString stringWithFormat:@"%@/sidebar-list-snapshot.json", e]];
        NSDate *m = [[NSFileManager defaultManager] attributesOfItemAtPath:p error:nil][NSFileModificationDate];
        if (m && (!bestDate || [m compare:bestDate] == NSOrderedDescending)) { bestDate = m; best = p; }
    }
    return best;
}

// 任务 30 秒刷一次：优先侧栏快照的 working 任务；没有再用最新会话的 ai-title 兜底
- (void)refreshCurrentTask {
    NSString *snap = [self latestSidebarSnapshot];
    NSString *p = [self latestTranscript];
    NSString *sid = p ? [[p lastPathComponent] stringByDeletingPathExtension] : nil;   // 最新会话 id
    NSString *t = snap ? [self taskTitleFromSidebarSnapshot:snap sessionId:sid] : nil;
    if (!t) {
        t = p ? [self latestTitleInFile:p] : nil;
    }
    if (!t) {
        NSString *p = [self latestTranscript];
        t = p ? [self latestTitleInFile:p] : nil;
    }
    self.currentTask = t;
}
// ================= Buddy 积分余额（V1.3，挂 W 旁边绿色数字） =================
// 数据源：POST https://www.codebuddy.cn/billing/meter/get-user-resource-summary
//   返回 data.TotalCredits 字符串（如 "5516.54000208"）＝客户端「积分余额」
// 凭据：复用积分助手技能的授权解析（buddy_station._resolve_auth，自动走本机登录态/缓存凭据），
//   ObjC 不碰 token —— 用 NSTask 跑一段内嵌 python 取数，只回传数字。
// 频率：60 秒一拉；失败保留上次真实值（宁旧勿假），从未成功就不显示。

// 找一个可用的 python3：系统自带优先，托管版本兜底
- (NSString *)firstAvailablePython {
    NSArray<NSString *> *candidates = @[@"/usr/bin/python3"];
    NSString *verDir = [NSHomeDirectory() stringByAppendingPathComponent:@".workbuddy/binaries/python/versions"];
    NSArray<NSString *> *vers = [[NSFileManager defaultManager] contentsOfDirectoryAtPath:verDir error:nil];
    for (NSString *v in [vers sortedArrayUsingComparator:^NSComparisonResult(NSString *a, NSString *b) {
        return [b compare:a options:NSNumericSearch];   // 新版本优先
    }])
        candidates = [candidates arrayByAddingObject:[verDir stringByAppendingPathComponent:[NSString stringWithFormat:@"%@/bin/python", v]]];
    for (NSString *p in candidates)
        if ([[NSFileManager defaultManager] isExecutableFileAtPath:p]) return p;
    return nil;
}

- (void)fetchCredits {
    if (self.creditFetching) return;
    NSString *py = [self firstAvailablePython];
    if (!py) return;
    NSString *skillDir = [NSHomeDirectory() stringByAppendingPathComponent:
        @".workbuddy/skills/totorosir-workbuddy-score/scripts"];
    if (![[NSFileManager defaultManager] fileExistsAtPath:skillDir]) return;
    self.creditFetching = YES;

    // 内嵌取数脚本：只打印两个数字（余额 / 周期已用累计），绝不打印 token
    // V1.9：第二个数 = 各 Packages 的 CycleUsedCapacity 求和——只增不减，签到/活动发分不影响它；
    // 余额（TotalCredits）会被每日签到 +100 等发分抬高，余额差值会被抵成 0（2026-10-26 实测踩坑）
    NSString *code = [NSString stringWithFormat:
        @"import sys,json,urllib.request\n"
         "sys.path.insert(0,r\"%1$@\")\n"
         "import buddy_station as bs\n"
         "tok,domain,_=bs._resolve_auth('auto')\n"
         "req=urllib.request.Request('https://www.codebuddy.cn/billing/meter/get-user-resource-summary',"
         "method='POST',data=b'{}',"
         "headers={'Authorization':'Bearer '+tok,'Content-Type':'application/json','User-Agent':'Mozilla/5.0'})\n"
         "d=json.load(urllib.request.urlopen(req,timeout=20))['data']\n"
         "print(d['TotalCredits'])\n"
         "print(sum(float(p['CycleUsedCapacity']) for p in d.get('Packages',[])))\n", skillDir];

    __weak typeof(self) weakSelf = self;
    dispatch_async(dispatch_get_global_queue(QOS_CLASS_UTILITY, 0), ^{
        NSTask *task = [[NSTask alloc] init];
        task.launchPath = py;
        task.arguments = @[@"-c", code];
        NSPipe *outPipe = [NSPipe pipe];
        task.standardOutput = outPipe;
        task.standardError = [NSFileHandle fileHandleWithNullDevice];
        __block typeof(self) blockSelf = weakSelf;
        task.terminationHandler = ^(NSTask *t) {
        NSData *d = [[outPipe fileHandleForReading] readDataToEndOfFile];
        NSString *text = [[NSString alloc] initWithData:d encoding:NSUTF8StringEncoding];
        text = [text stringByTrimmingCharactersInSet:[NSCharacterSet whitespaceAndNewlineCharacterSet]];
        NSArray *lines = [text componentsSeparatedByString:@"\n"];
        NSString *balStr = lines.firstObject ?: @"";
        NSString *usedStr = (lines.count > 1) ? lines[1] : @"";
        NSInteger val = -1;
        double raw = -1, usedRaw = -1;
        if (balStr.length && balStr.doubleValue >= 0) {
            raw = balStr.doubleValue;                     // 5516.540002 原始浮点
            val = (NSInteger)llround(raw);                // 5516.54 → 5517（菜单栏显示用）
        }
        if (usedStr.length && usedStr.doubleValue >= 0) usedRaw = usedStr.doubleValue;
        dispatch_async(dispatch_get_main_queue(), ^{
            blockSelf.creditFetching = NO;
            if (getenv("WBLIGHT_DEBUG"))
                NSLog(@"[credits] val=%ld (raw=%@ used=%@)", (long)val, balStr, usedStr);
            if (raw >= 0) {
                blockSelf.creditsRaw = raw;
                [blockSelf updateSpentWithRaw:raw used:usedRaw];   // 今日消耗 = 当日已用累计 − 当前已用累计
            }
            if (val >= 0 && val != blockSelf.credits) {
                blockSelf.credits = val;
                [blockSelf render];                       // 只在数字变了时重绘，避免闪烁抖动
            }
        });
        };
        [task launch];
    });
}

// ================= 今日消耗（V1.9，已用累计差值法；V1.5 为余额差值法） =================
// 官方接口（get-user-resource-summary）只有周期累计，没有按天明细（已探测过一批端点，全 404）。
// V1.5 用「当日余额基线 − 当前余额」：被每日签到 +100 等发分抬高余额后差值变负 → 恒显示 0（2026-10-26 实测踩坑）。
// V1.9 改用「当日已用累计基线 − 当前已用累计」：CycleUsedCapacity 只增不减、发分不影响它，才是真消耗。
// 基线存盘 ~/Library/Application Support/WBLight/spent_day.txt（V1.9 起三段：日期 余额 已用累计，兼容旧两段）。

- (NSString *)spentStorePath {
    NSString *dir = [NSHomeDirectory() stringByAppendingPathComponent:@"Library/Application Support/WBLight"];
    [[NSFileManager defaultManager] createDirectoryAtPath:dir withIntermediateDirectories:YES attributes:nil error:nil];
    return [dir stringByAppendingPathComponent:@"spent_day.txt"];
}

- (NSString *)todayKey {
    static NSDateFormatter *fmt = nil;
    static dispatch_once_t once;
    dispatch_once(&once, ^{ fmt = [[NSDateFormatter alloc] init]; fmt.dateFormat = @"yyyy-MM-dd"; });
    return [fmt stringFromDate:[NSDate date]];
}

- (void)loadSpentBaseline {
    NSString *s = [NSString stringWithContentsOfFile:self.spentStorePath
                                            encoding:NSUTF8StringEncoding error:nil];
    NSArray *parts = [s componentsSeparatedByString:@" "];
    if (parts.count >= 2 && [parts[0] isKindOfClass:[NSString class]]
        && [parts[0] isEqualToString:[self todayKey]]) {
        self.spentDayKey = parts[0];
        self.spentBaseline = [(NSString *)parts[1] doubleValue];
        self.spentBaselineValid = (self.spentBaseline > 0);
        self.usedBaseline = (parts.count >= 3) ? [(NSString *)parts[2] doubleValue] : -1;  // 旧两段文件 = 未知
        self.todaySpent = 0;   // 启动时还没新数据，先 0，等第一次拉到余额再算
    }
}

- (void)saveSpentBaseline {
    NSString *s = [NSString stringWithFormat:@"%@ %.6f %.6f", self.spentDayKey,
                   self.spentBaseline, self.usedBaseline];
    [s writeToFile:self.spentStorePath atomically:YES encoding:NSUTF8StringEncoding error:nil];
}

- (void)updateSpentWithRaw:(double)raw used:(double)usedRaw {
    NSTimeInterval now = [[NSDate date] timeIntervalSince1970];
    if (![self.todayKey isEqualToString:self.spentDayKey] || !self.spentBaselineValid) {
        self.spentDayKey = [self todayKey];   // 新的一天：以当前余额/已用累计为基线
        self.spentBaseline = raw;
        self.usedBaseline = (usedRaw >= 0) ? usedRaw : -1;
        self.spentBaselineValid = YES;
        self.todaySpent = 0;
        self.lastSpentCalc = now;
        [self saveSpentBaseline];
        return;
    }
    if (now - self.lastSpentCalc < kSpentRefresh) return;   // 10 分钟节流：不重要指标，别太频繁
    self.lastSpentCalc = now;
    if (usedRaw >= 0) {
        if (self.usedBaseline < 0) {              // 旧格式文件没存已用锚点：本次补上，从下次开始准
            self.usedBaseline = usedRaw;
            [self saveSpentBaseline];
            return;
        }
        double spent = usedRaw - self.usedBaseline;
        if (spent < 0) {                          // 周期翻转/包过期会让已用回落：重置锚点，不显示负数
            self.usedBaseline = usedRaw;
            [self saveSpentBaseline];
            self.todaySpent = 0;
            return;
        }
        self.todaySpent = spent;
        return;
    }
    double spent = self.spentBaseline - raw;      // 兜底：接口没给已用量时退回余额差值
    self.todaySpent = spent > 0 ? spent : 0;   // 余额变大（充值/活动补发）不倒扣，宁少勿假
}

// 消耗数字格式：0.05 以下记 0；整数不带小数；否则留 1 位小数
+ (NSString *)formatSpent:(double)v {
    if (v < 0.05) return @"0";
    if (fabs(v - round(v)) < 0.05) return [NSString stringWithFormat:@"%.0f", v];
    return [NSString stringWithFormat:@"%.1f", v];
}

// ================= 判定（纯函数，便于自测） =================

// 在线 且（回合在途 或 [仅当无 transcript 时] 心跳新鲜）→ 忙
// 关键：只要 transcript 可用，就以"回合是否走完"为唯一判据 —— 交付即红，秒级，
// 不再被审计日志的兜底窗口拖出"干完活还绿闪几十秒"的尾巴。
- (BOOL)isBusyOnline:(BOOL)online heart:(NSTimeInterval)heart
            inFlight:(BOOL)inFlight inFlightSince:(NSTimeInterval)since
       hasTranscript:(BOOL)hasTranscript
                 now:(NSTimeInterval)now window:(NSTimeInterval)w {
    if (!online) return NO;                          // 没开 WorkBuddy → 一律红
    if (hasTranscript)
        return inFlight && since > 0 && (now - since) < kMaxInFlight;  // 在途→绿，交付→红
    if (heart <= 0) return NO;                       // 无 transcript 且无心跳 → 红
    return (now - heart) < w;                        // 兜底：心跳窗口内 → 绿
}

// ================= 状态刷新与渲染 =================

- (void)refresh {
    NSTimeInterval now = [[NSDate date] timeIntervalSince1970];
    self.online = [self workBuddyRunning];

    NSDictionary *ts = [self transcriptState];
    NSTimeInterval tHeart = ts ? [ts[@"time"] doubleValue] : 0;
    BOOL tInFlight = ts ? [ts[@"inFlight"] boolValue] : NO;
    self.transcriptEvent = ts ? ts[@"name"] : nil;
    if (now - self.lastTaskFetch >= kTaskRefresh) {   // 任务 30 秒刷一次，不跟 1 秒巡检抢开销
        self.lastTaskFetch = now;
        [self refreshCurrentTask];
    }

    self.lastActivity = [self lastActivityTime];                 // 审计日志心跳（兜底）
    self.heartTime = MAX(tHeart, self.lastActivity);             // 双源取新
    self.inFlight = tInFlight;
    self.inFlightSince = tHeart;
    self.hasTranscript = (ts != nil);

    BOOL wasBusy = self.busy;
    self.busy = [self isBusyOnline:self.online heart:self.heartTime
                           inFlight:self.inFlight inFlightSince:self.inFlightSince
                      hasTranscript:self.hasTranscript
                               now:now window:kBusyWindow]
             || [self snapshotHasWorkingTask];   // V2.2：快照里有 ✳ 任务就不灭灯（刚哥定）
    if (self.busy && !wasBusy) self.busySince = now;         // 记下"这次忙了多久"
    if (!self.busy) self.busySince = 0;
    if (wasBusy && !self.busy) [self playCompletionSound];   // 绿 → 红：活干完了，叮一声（V1.8）
    self.lastAction = [self readLastAction];
    self.todayCalls = [self todayCallCount];

    if (self.busy) [self startBlink]; else { [self stopBlink]; self.blinkOn = YES; }
    [self render];
    [self renderMenu];
}

- (void)startBlink {
    if (self.blinkTimer) return;
    self.blinkTimer = [NSTimer timerWithTimeInterval:kBlinkInterval target:self
                                            selector:@selector(tickBlink) userInfo:nil repeats:YES];
    [[NSRunLoop mainRunLoop] addTimer:self.blinkTimer forMode:NSRunLoopCommonModes];
}

- (void)stopBlink {
    [self.blinkTimer invalidate];
    self.blinkTimer = nil;
}

- (void)tickBlink {
    self.blinkOn = !self.blinkOn;
    [self render];
}

// 顶条：绿色加粗闪烁 W（忙） / 红色加粗常亮 W（闲/离线）
- (void)render {
    NSColor *base = self.busy ? [NSColor colorWithCalibratedRed:0.24 green:0.95 blue:0.42 alpha:1.0]
                              : [NSColor colorWithCalibratedRed:1.0  green:0.28 blue:0.24 alpha:1.0];
    NSColor *col = (self.busy && !self.blinkOn) ? [base colorWithAlphaComponent:0.22] : base;
    NSMutableAttributedString *as = [[NSMutableAttributedString alloc] initWithString:@"W"];
    NSRange all = NSMakeRange(0, as.length);
    [as addAttribute:NSFontAttributeName
               value:[NSFont systemFontOfSize:15 weight:NSFontWeightHeavy] range:all];
    [as addAttribute:NSForegroundColorAttributeName value:col range:all];
    // W 旁边挂积分余额：绿色数字，不闪烁，与忙闲无关（V1.3）
    if (self.credits >= 0) {
        NSString *num = [NSString stringWithFormat:@" %ld", (long)self.credits];
        NSRange r = NSMakeRange(as.length, num.length);
        [as appendAttributedString:[[NSAttributedString alloc] initWithString:num]];
        [as addAttribute:NSFontAttributeName
                   value:[NSFont systemFontOfSize:12 weight:NSFontWeightSemibold] range:r];
        [as addAttribute:NSForegroundColorAttributeName
                   value:[NSColor colorWithCalibratedRed:0.30 green:0.95 blue:0.45 alpha:1.0] range:r];
    }
    self.item.button.attributedTitle = as;
    self.item.button.toolTip = self.busy ? @"WorkBuddy 正在干活" : @"WorkBuddy 闲置 / 未运行";
}

- (NSTextField *)makeField:(CGFloat)size bold:(BOOL)bold {
    NSTextField *tf = [[NSTextField alloc] initWithFrame:NSMakeRect(14, 2, 286, 17)];
    tf.editable = NO; tf.bordered = NO; tf.drawsBackground = NO;
    tf.font = [NSFont monospacedDigitSystemFontOfSize:size
                                               weight:(bold ? NSFontWeightSemibold : NSFontWeightRegular)];
    tf.textColor = [NSColor labelColor];
    tf.lineBreakMode = NSLineBreakByTruncatingTail;
    return tf;
}

- (NSMenuItem *)rowForField:(NSTextField *)f {
    NSView *v = [[NSView alloc] initWithFrame:NSMakeRect(0, 0, 314, 21)];
    [v addSubview:f];
    NSMenuItem *mi = [[NSMenuItem alloc] init];
    mi.view = v;
    return mi;
}

- (NSAttributedString *)text:(NSString *)str color:(NSColor *)color size:(CGFloat)size bold:(BOOL)bold {
    NSMutableAttributedString *as = [[NSMutableAttributedString alloc] initWithString:str];
    NSRange all = NSMakeRange(0, as.length);
    [as addAttribute:NSFontAttributeName
               value:[NSFont monospacedDigitSystemFontOfSize:size
                                                      weight:(bold ? NSFontWeightSemibold : NSFontWeightRegular)]
               range:all];
    [as addAttribute:NSForegroundColorAttributeName value:color range:all];
    NSMutableParagraphStyle *ps = [NSMutableParagraphStyle new];
    ps.lineBreakMode = NSLineBreakByTruncatingTail;
    [as addAttribute:NSParagraphStyleAttributeName value:ps range:all];
    return as;
}

- (NSString *)agoText:(NSTimeInterval)ts {
    if (ts <= 0) return @"无记录";
    NSTimeInterval d = [[NSDate date] timeIntervalSince1970] - ts;
    if (d < 0) d = 0;
    if (d < 60)   return [NSString stringWithFormat:@"%.0f 秒前", d];
    if (d < 3600) return [NSString stringWithFormat:@"%.0f 分钟前", d / 60];
    return [NSString stringWithFormat:@"%.1f 小时前", d / 3600];
}

- (void)renderMenu {
    // 状态行（第二行，V2.0）：文字白色、与其他行同字号；只有「已忙」时长的两个数字绿色（2026-10-26 刚哥指定）
    if (!self.online || !self.busy) {
        NSString *st = self.online ? @"闲置中" : @"WorkBuddy 未运行";
        self.statusField.attributedStringValue = [self text:st color:[NSColor labelColor] size:11 bold:NO];
    } else {
        double secs = [[NSDate date] timeIntervalSince1970] - self.busySince;
        long mins = (long)floor(secs / 60), secs2 = (long)fmod(secs, 60);
        NSString *pre = @"工作中 · 已忙 ", *mid1 = @" 分 ", *mid2 = @" 秒";
        NSMutableAttributedString *sa = [[NSMutableAttributedString alloc]
            initWithString:[NSString stringWithFormat:@"%@%ld%@%ld%@", pre, mins, mid1, secs2, mid2]
                attributes:@{NSFontAttributeName: self.statusField.font,
                             NSForegroundColorAttributeName: [NSColor labelColor]}];
        NSString *ms = [NSString stringWithFormat:@"%ld", mins], *ss = [NSString stringWithFormat:@"%ld", secs2];
        [sa addAttribute:NSForegroundColorAttributeName value:[NSColor systemGreenColor]
                    range:NSMakeRange(pre.length, ms.length)];
        [sa addAttribute:NSForegroundColorAttributeName value:[NSColor systemGreenColor]
                    range:NSMakeRange(pre.length + ms.length + mid1.length, ss.length)];
        self.statusField.attributedStringValue = sa;
    }
    // 任务行（第一行，V2.0）：整行绿字、大一号（13 半粗）（2026-10-26 刚哥指定）
    NSString *task = self.currentTask.length ? self.currentTask : @"—";
    self.taskField.attributedStringValue = [self text:[NSString stringWithFormat:@"当前任务：%@", task]
                                                color:[NSColor systemGreenColor] size:15 bold:YES];
    // 今日调用（V1.6）：次数数字染绿（2026-10-26 刚哥指定），其余保持次要灰、不变色
    {
        NSString *cPre = @"今日调用  ", *cMid = @" 次   ·   活动于 ", *ago = [self agoText:self.lastActivity];
        NSMutableAttributedString *ca = [[NSMutableAttributedString alloc]
            initWithString:[NSString stringWithFormat:@"%@%ld%@%@",
                                cPre, (long)self.todayCalls, cMid, ago ?: @"—"]
                attributes:@{NSFontAttributeName: self.countField.font,
                             NSForegroundColorAttributeName: [NSColor secondaryLabelColor]}];
        [ca addAttribute:NSForegroundColorAttributeName
                   value:[NSColor systemGreenColor]
                   range:NSMakeRange(cPre.length, [NSString stringWithFormat:@"%ld", (long)self.todayCalls].length)];
        self.countField.attributedStringValue = ca;
    }
    // 今日消耗（V1.9）：已用累计差值，10 分钟刷一次（2026-10-26 刚哥改为 10 分钟）；数字红色，
    // 尾缀「（10 分钟更新）」白色小字提示刷新节奏（刚哥指定），其余保持次要灰
    NSString *spent = @"—";
    if (self.spentBaselineValid && [self.spentDayKey isEqualToString:[self todayKey]])
        spent = [Delegate formatSpent:self.todaySpent];
    NSString *pre = @"今日消耗  ", *mid = @" 积分", *suf = @"（10 分钟更新）";
    NSMutableAttributedString *sa = [[NSMutableAttributedString alloc]
        initWithString:[NSString stringWithFormat:@"%@%@%@%@", pre, spent, mid, suf]
            attributes:@{NSFontAttributeName: self.spentField.font,
                         NSForegroundColorAttributeName: [NSColor secondaryLabelColor]}];
    NSRange numRange = NSMakeRange(pre.length, spent.length);
    NSRange sufRange = NSMakeRange(pre.length + spent.length + mid.length, suf.length);
    if (![spent isEqualToString:@"—"])   // 数字红（与灯语红同色）
        [sa addAttribute:NSForegroundColorAttributeName
                   value:[NSColor colorWithCalibratedRed:1.0 green:0.30 blue:0.28 alpha:1.0]
                   range:numRange];
    [sa addAttribute:NSForegroundColorAttributeName value:[NSColor labelColor] range:sufRange];   // 白字
    self.spentField.attributedStringValue = sa;
}

// ================= 菜单 =================

- (void)buildMenu {
    self.menu = [[NSMenu alloc] init];
    self.menu.autoenablesItems = NO;
    self.menu.delegate = self;

    self.statusField = [self makeField:11 bold:NO];    // V2.0：与其他行同字号（刚哥指定）
        self.taskField   = [self makeField:15 bold:YES];   // V2.2：与 Codex小灯 统一 15 号（按大的统一，刚哥指定）
    self.taskField.frame = NSMakeRect(14, 1, 286, 19);   // 15 号字需要更高，防截字
    self.countField  = [self makeField:11 bold:NO];
    self.spentField  = [self makeField:11 bold:NO];
    for (NSTextField *f in @[self.taskField, self.statusField])   // V2.3：任务、状态两行（最近动作行已删）
        [self.menu addItem:[self rowForField:f]];
    [self.menu addItem:[NSMenuItem separatorItem]];   // V2.0：把「今日调用/今日消耗」两个统计行隔开（2026-10-26 刚哥指定）
    for (NSTextField *f in @[self.countField, self.spentField])
        [self.menu addItem:[self rowForField:f]];

    [self.menu addItem:[NSMenuItem separatorItem]];
    NSMenuItem *open = [[NSMenuItem alloc] initWithTitle:@"打开 WorkBuddy" action:@selector(openWB:) keyEquivalent:@""];
    open.target = self;
    [self.menu addItem:open];

    // ---- 更换提示音（V1.8，整套移植自 Codex小灯：开关 + 音源分组 + 试听；刚哥指定放在「打开 WorkBuddy」下面）----
    NSMenuItem *soundRoot = [[NSMenuItem alloc] initWithTitle:@"更换提示音" action:nil keyEquivalent:@""];
    NSMenu *soundMenu = [[NSMenu alloc] initWithTitle:@"更换提示音"];

    self.soundToggleItem = [[NSMenuItem alloc] initWithTitle:@"提示音：开启"
                                                       action:@selector(toggleSound:)
                                                keyEquivalent:@""];
    self.soundToggleItem.target = self;
    [soundMenu addItem:self.soundToggleItem];
    [soundMenu addItem:[NSMenuItem separatorItem]];

    self.soundChoiceItems = [NSMutableArray array];
    NSString *lastGroup = nil;
    for (NSDictionary<NSString *, NSString *> *s in [self availableSounds]) {
        NSString *group = s[@"group"];
        if (![group isEqualToString:lastGroup ?: @""]) {
            NSMenuItem *h = [[NSMenuItem alloc] initWithTitle:group action:nil keyEquivalent:@""];
            h.enabled = NO;
            [soundMenu addItem:h];
            lastGroup = group;
        }
        NSMenuItem *it = [[NSMenuItem alloc] initWithTitle:s[@"name"]
                                                    action:@selector(selectSound:)
                                             keyEquivalent:@""];
        it.target = self;
        it.representedObject = s[@"path"];
        [self.soundChoiceItems addObject:it];
        [soundMenu addItem:it];
    }
    [soundMenu addItem:[NSMenuItem separatorItem]];
    NSMenuItem *preview = [[NSMenuItem alloc] initWithTitle:@"试听当前提示音"
                                                     action:@selector(previewSound:)
                                              keyEquivalent:@""];
    preview.target = self;
    [soundMenu addItem:preview];

    soundRoot.submenu = soundMenu;
    [self.menu addItem:soundRoot];
    [self refreshSoundMenuItems];

    [self.menu addItem:[NSMenuItem separatorItem]];
    NSMenuItem *restart = [[NSMenuItem alloc] initWithTitle:@"重启 WB 小灯" action:@selector(restartApp:) keyEquivalent:@""];
    restart.target = self;
    // 左侧配一个「循环箭头」图标：既填充空档，又和「退出」的图标位对齐（2026-10-25 刚哥指定）
    NSImage *sym = [NSImage imageWithSystemSymbolName:@"arrow.triangle.2.circlepath"
                               accessibilityDescription:@"重启"];
    if (sym) {
        restart.image = [sym imageWithSymbolConfiguration:
            [NSImageSymbolConfiguration configurationWithPointSize:13 weight:NSFontWeightMedium]];
    }
    [self.menu addItem:restart];
    NSMenuItem *quit = [[NSMenuItem alloc] initWithTitle:@"退出WB小灯" action:@selector(terminate:) keyEquivalent:@"q"];   // V1.7：原名「退出」太泛（2026-10-26 刚哥指定）
    // V2.4：退出补上「方框叉」图标（xmark.square）——截图里一直想要的样子，但源码里从没写过，这次落盘（2026-10-27 刚哥指定）
    NSImage *quitSym = [NSImage imageWithSystemSymbolName:@"xmark.square"
                                   accessibilityDescription:@"退出"];
    if (quitSym) {
        quit.image = [quitSym imageWithSymbolConfiguration:
            [NSImageSymbolConfiguration configurationWithPointSize:13 weight:NSFontWeightMedium]];
    }
    [self.menu addItem:quit];
    self.item.menu = self.menu;
}

// 重启自己：与 Codex小灯 同一套修好的逻辑——
// 1) createsNewApplicationInstance 必须设 YES，否则系统认为"已在运行"不会开新的；
// 2) 不能发起启动后立刻 terminate（请求是异步的，进程一死请求被掐掉，表现就是"点了没反应"），
//    必须等 completionHandler 确认新实例拉起后再延迟退出；失败走 /usr/bin/open -n 兜底。（2026-10-25）
- (void)restartApp:(id)sender {
    NSURL *selfURL = [NSURL fileURLWithPath:[[NSBundle mainBundle] bundlePath]];
    NSWorkspaceOpenConfiguration *conf = [NSWorkspaceOpenConfiguration configuration];
    conf.createsNewApplicationInstance = YES;
    [[NSWorkspace sharedWorkspace] openApplicationAtURL:selfURL
                                          configuration:conf
                                      completionHandler:^(NSRunningApplication *app, NSError *error) {
        dispatch_async(dispatch_get_main_queue(), ^{
            if (error || !app) {
                NSTask *task = [[NSTask alloc] init];
                task.launchPath = @"/usr/bin/open";
                task.arguments = @[@"-n", selfURL.path];
                [task launch];
            }
            dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(0.6 * NSEC_PER_SEC)),
                           dispatch_get_main_queue(), ^{
                [NSApp terminate:nil];
            });
        });
    }];
}

- (void)openWB:(id)sender {
    NSURL *u = [self workBuddyAppURL];
    if (u) [[NSWorkspace sharedWorkspace] openApplicationAtURL:u
                                                configuration:[NSWorkspaceOpenConfiguration configuration]
                                            completionHandler:nil];
}

// 重放真实 transcript：用真实代码逻辑逐行算忙闲，验证"思考→绿 / 交付→红"
- (int)replayTranscript {
    NSString *p = [self latestTranscript];
    if (!p) { printf("找不到 transcript\n"); return 1; }
    NSString *text = [NSString stringWithContentsOfFile:p encoding:NSUTF8StringEncoding error:nil];
    if (!text.length) { printf("transcript 读不出内容\n"); return 1; }
    printf("重放文件：%s\n\n", p.UTF8String);

    NSInteger nGreen = 0, nRed = 0, nFlips = 0, nReason = 0, nDone = 0, nSnapSkip = 0, nTotal = 0;
    BOOL haveLast = NO, lastBusy = NO, lastWasDone = NO;
    NSMutableArray<NSString *> *flips = [NSMutableArray array];
    for (NSString *line in [text componentsSeparatedByString:@"\n"]) {
        NSString *t = [line stringByTrimmingCharactersInSet:
                           [NSCharacterSet whitespaceAndNewlineCharacterSet]];
        if (t.length < 2) continue;
        NSDictionary *e = [NSJSONSerialization JSONObjectWithData:[t dataUsingEncoding:NSUTF8StringEncoding]
                                                         options:0 error:nil];
        if (![e isKindOfClass:[NSDictionary class]] || !e.count) continue;
        nTotal++;
        if (![self isSubstantiveEvent:e]) { nSnapSkip++; continue; }   // 被跳过的记账事件
        NSString *name = [self eventName:e];
        BOOL busy = [self eventInFlight:e];
        if (busy) nGreen++; else nRed++;
        if ([name isEqualToString:@"深度思考中"]) nReason++;
        BOOL isDone = [name isEqualToString:@"已交付回复"];
        if (isDone) nDone++;
        if (haveLast && busy != lastBusy) {
            nFlips++;
            [flips addObject:[NSString stringWithFormat:@"%@ → %@   （%@）",
                              lastBusy ? @"绿" : @"红", busy ? @"绿" : @"红", name]];
        }
        // 关键回归：每次"已交付"都必须判红，且是紧接着发生的（0 延迟）
        if (isDone && busy) printf("  ⚠️ 已交付却被判为在途！行号 %ld\n", (long)nTotal);
        if (lastWasDone && busy) { /* 交付后立刻又接新活，正常 */ }
        haveLast = YES; lastBusy = busy; lastWasDone = isDone;
    }
    printf("总行数 %ld\n", (long)nTotal);
    printf("  实质事件 绿=%ld 红=%ld   （记账类跳过 %ld 行）\n",
           (long)nGreen, (long)nRed, (long)nSnapSkip);
    printf("  深度思考行 %ld · 交付事件 %ld · 状态翻转 %ld 次\n\n",
           (long)nReason, (long)nDone, (long)nFlips);
    printf("最近 16 次灯色翻转（真实数据）：\n");
    NSInteger start = MAX(0, (NSInteger)flips.count - 16);
    for (NSInteger i = start; i < (NSInteger)flips.count; i++)
        printf("  %s\n", flips[(NSUInteger)i].UTF8String);
    NSInteger bad = 0;
    // 交付即红：nDone 次交付里，不该有任何一次被判在途
    for (NSString *line in [text componentsSeparatedByString:@"\n"]) {
        NSString *t = [line stringByTrimmingCharactersInSet:
                           [NSCharacterSet whitespaceAndNewlineCharacterSet]];
        if (t.length < 2) continue;
        NSDictionary *e = [NSJSONSerialization JSONObjectWithData:[t dataUsingEncoding:NSUTF8StringEncoding]
                                                         options:0 error:nil];
        if (![e isKindOfClass:[NSDictionary class]] || !e.count) continue;
        if ([self isSubstantiveEvent:e] && [self eventInFlight:e]) {
            NSString *type = [self strField:e key:@"type"];
            NSString *role = [self strField:e key:@"role"];
            NSString *stat = [self strField:e key:@"status"];
            if ([type isEqualToString:@"message"] && [role isEqualToString:@"assistant"]
                && [stat isEqualToString:@"completed"]) bad++;
        }
    }
    printf("\n结论：%s\n", bad == 0 ? "「已交付」全部正确判红，无假绿 ✅" : "存在假绿 ❌");
    return bad == 0 ? 0 : 1;
}

// 一行状态（外部采样用）：时间 灯色 在线 在途 末事件
- (NSString *)stateLine {
    NSDictionary *live = [self transcriptState];
    BOOL online = [self workBuddyRunning];
    NSTimeInterval heart = [self lastActivityTime];
    NSTimeInterval ts = live ? [live[@"time"] doubleValue] : 0;
    BOOL inFlight = live ? [live[@"inFlight"] boolValue] : NO;
    BOOL busy = [self isBusyOnline:online heart:(live ? MAX(ts, heart) : heart)
                          inFlight:inFlight inFlightSince:ts
                     hasTranscript:(live != nil)
                               now:[[NSDate date] timeIntervalSince1970] window:kBusyWindow];
    NSDateFormatter *df = [[NSDateFormatter alloc] init];
    df.dateFormat = @"HH:mm:ss";
    return [NSString stringWithFormat:@"%@   %@  在线=%@ 在途=%@  末事件=%@",
            [df stringFromDate:[NSDate date]],
            busy ? @"🟢绿" : @"🔴红",
            online ? @"是" : @"否", inFlight ? @"是" : @"否",
            [self strField:(live ?: @{}) key:@"name"].length ? live[@"name"] : self.lastAction];
}

// ================= 提示音（V1.8，整套移植自 Codex小灯 同款实现） =================
// 触发：绿 → 红（忙完）时「叮」一声；可开关、可换音、可试听；选择存 NSUserDefaults（suite: io.wblight.menubar）

- (NSUserDefaults *)soundPrefs {
    // V2.6 真修（2026-10-28 打包前用 .app 实测查出）：
    // 以前用 initWithSuiteName:@"io.wblight.menubar"，而这个 app 的 Bundle ID 恰好也叫这个名字，
    // 系统直接告警 "Using your own bundle identifier as an NSUserDefaults suite name ... will not work"，
    // 返回的实例不落盘、只活在进程内存里 → 换完提示音重启就丢（真 bug）。
    // 更坑的是：自测跑的是没打进 .app 的裸二进制（没有 Bundle ID，suite 正常）→ 自测一直 PASS，
    // 属于典型的"测试环境和真实运行环境不一致，把人骗了"。
    // 正解：用 standardUserDefaults —— 写的就是 ~/Library/Preferences/io.wblight.menubar.plist，
    // 两个 app 的 Bundle ID 不同，天然互不干扰。
    return [NSUserDefaults standardUserDefaults];
}

- (void)loadSoundPrefs {
    NSUserDefaults *d = [self soundPrefs];
    self.soundEnabled = ([d objectForKey:@"SoundEnabled"] == nil) ? YES : [d boolForKey:@"SoundEnabled"];
    NSString *p = [d stringForKey:@"SoundPath"];
    self.soundPath = p.length ? p : [self defaultSoundPath];
}

- (void)saveSoundPrefs {
    NSUserDefaults *d = [self soundPrefs];
    [d setBool:self.soundEnabled forKey:@"SoundEnabled"];
    [d setObject:(self.soundPath ?: @"") forKey:@"SoundPath"];
    [d synchronize];   // V2.3：立刻落盘，别等系统的异步 flush
}

- (NSString *)defaultSoundPath {
    // 默认就是那声「叮」；万一系统里没有，退回列表第一个
    NSString *tink = @"/System/Library/Sounds/Tink.aiff";
    if ([[NSFileManager defaultManager] fileExistsAtPath:tink]) return tink;
    NSDictionary<NSString *, NSString *> *first = [self availableSounds].firstObject;
    return first[@"path"] ?: @"";
}

- (NSArray<NSDictionary<NSString *, NSString *> *> *)availableSounds {
    NSArray<NSArray<NSString *> *> *dirs = @[
        @[@"系统提示音", @"/System/Library/Sounds"],
        @[@"其他提示音", @"/Library/Sounds"],
        @[@"自定义（~/Library/Sounds）", [NSHomeDirectory() stringByAppendingPathComponent:@"Library/Sounds"]],
    ];
    NSArray<NSString *> *exts = @[@"aiff", @"aif", @"wav", @"caf", @"m4a", @"mp3"];
    NSMutableArray<NSDictionary<NSString *, NSString *> *> *out = [NSMutableArray array];
    NSMutableSet<NSString *> *seen = [NSMutableSet set];
    NSFileManager *fm = [NSFileManager defaultManager];
    for (NSArray<NSString *> *pair in dirs) {
        NSString *group = pair[0];
        NSString *dir = pair[1];
        NSArray<NSString *> *names = [fm contentsOfDirectoryAtPath:dir error:nil];
        for (NSString *n in [names sortedArrayUsingSelector:@selector(localizedStandardCompare:)]) {
            if ([n hasPrefix:@"."]) continue;
            if (![exts containsObject:n.pathExtension.lowercaseString]) continue;
            NSString *base = n.stringByDeletingPathExtension;
            if ([seen containsObject:base]) continue;      // 同名只留第一个（系统音优先）
            [seen addObject:base];
            [out addObject:@{@"name": base,
                             @"path": [dir stringByAppendingPathComponent:n],
                             @"group": group}];
        }
    }
    return out;
}

- (void)refreshSoundMenuItems {
    self.soundToggleItem.title = self.soundEnabled ? @"提示音：开启（点一下关闭）"
                                                   : @"提示音：关闭（点一下开启）";
    self.soundToggleItem.state = self.soundEnabled ? NSControlStateValueOn : NSControlStateValueOff;
    NSString *cur = self.soundPath ?: @"";
    for (NSMenuItem *it in self.soundChoiceItems) {
        id rep = it.representedObject;
        NSString *p = [rep isKindOfClass:[NSString class]] ? rep : @"";
        it.state = [p isEqualToString:cur] ? NSControlStateValueOn : NSControlStateValueOff;
    }
}

- (void)playCompletionSound {
    if (!self.soundEnabled || !self.soundPath.length) return;
    if (![[NSFileManager defaultManager] fileExistsAtPath:self.soundPath]) return;
    NSTimeInterval now = [NSDate date].timeIntervalSince1970;
    if (self.lastSoundAt > 0 && now - self.lastSoundAt < kMinSoundGap) return;  // 防抖，避免连响
    self.lastSoundAt = now;
    NSSound *s = [[NSSound alloc] initWithContentsOfFile:self.soundPath byReference:YES];
    if (!s) return;
    self.player = s;              // 必须持有，否则声音会被提前释放（只响半声或干脆不响）
    [s play];
}

- (void)toggleSound:(id)sender {
    self.soundEnabled = !self.soundEnabled;
    [self saveSoundPrefs];
    [self refreshSoundMenuItems];
    if (self.soundEnabled) {
        self.lastSoundAt = 0;     // 开启时立刻响一声，马上知道听不听得到
        [self playCompletionSound];
    }
}

- (void)selectSound:(id)sender {
    if (![sender isKindOfClass:[NSMenuItem class]]) return;
    id rep = [(NSMenuItem *)sender representedObject];
    if (![rep isKindOfClass:[NSString class]] || ![(NSString *)rep length]) return;
    self.soundPath = rep;
    self.soundEnabled = YES;       // 选了音就等于要开，符合直觉
    [self saveSoundPrefs];
    [self refreshSoundMenuItems];
    self.lastSoundAt = 0;
    [self playCompletionSound];    // 点一下即试听
}

- (void)previewSound:(id)sender {
    BOOL keep = self.soundEnabled;
    self.soundEnabled = YES;
    self.lastSoundAt = 0;
    [self playCompletionSound];
    self.soundEnabled = keep;
}

// ================= 启动 =================

- (void)applicationDidFinishLaunching:(NSNotification *)n {
    self.item = [[NSStatusBar systemStatusBar] statusItemWithLength:NSVariableStatusItemLength];
    [self loadSoundPrefs];                   // 提示音偏好（V1.8）—— 必须在 buildMenu 之前，勾选态才对
    [self buildMenu];
    self.blinkOn = YES;
    self.credits = -1;                       // -1 = 还没拿到，先只显示 W
    [self loadSpentBaseline];                // 今日消耗基线（V1.5，跨启动持久化）
    [self refresh];
    self.pollTimer = [NSTimer timerWithTimeInterval:kPollInterval target:self
                                            selector:@selector(refresh) userInfo:nil repeats:YES];
    [[NSRunLoop mainRunLoop] addTimer:self.pollTimer forMode:NSRunLoopCommonModes];
    // 积分余额：启动后 2 秒首拉，之后 60 秒一次（V1.3）
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2.0 * NSEC_PER_SEC)),
                   dispatch_get_main_queue(), ^{ [self fetchCredits]; });
    self.creditTimer = [NSTimer timerWithTimeInterval:60.0 target:self
                                            selector:@selector(fetchCredits) userInfo:nil repeats:YES];
    self.creditTimer.tolerance = 5.0;
    [[NSRunLoop mainRunLoop] addTimer:self.creditTimer forMode:NSRunLoopCommonModes];
}

// ================= 自测 =================

- (int)runSelfTest {
    __block int fails = 0;
    void (^check)(BOOL, NSString *, NSString *) = ^(BOOL ok, NSString *name, NSString *detail) {
        printf("%s  %s%s\n", ok ? "PASS" : "FAIL", name.UTF8String,
               detail ? [NSString stringWithFormat:@"  (%@)", detail].UTF8String : "");
        if (!ok) fails++;
    };

    printf("== 1. 心跳源：只认工具调用日志 ==\n");
    NSString *sp = [self newestSpoolFile];
    check(sp.length > 0, @"spool 有日志文件", sp ? sp.lastPathComponent : @"未找到");
    NSDictionary *ev = sp ? [self tailJSONOf:sp] : nil;
    check(ev.count > 0, @"tailJSONOf 能解析出最后一行事件", ev[@"eventType"]);
    check([self secondsFromEvent:ev] > 0, @"事件里有可用 timestamp",
          [self agoText:[self secondsFromEvent:ev]]);
    check([self lastActivityTime] > 0, @"合并心跳 > 0", [self agoText:[self lastActivityTime]]);
    check([[self todayArchiveFile] hasSuffix:@".jsonl"], @"当天归档路径可构造",
          [[self todayArchiveFile] lastPathComponent]);

    printf("== 2. 误报源剔除（V1.1 修正）==\n");
    // changes-index 会在用户打字/发消息时写入，比真实动作更新；心跳不能被它带跑
    NSTimeInterval idxM = [self latestMTimeIn:[[self wbHome] stringByAppendingPathComponent:@"changes-index"]];
    NSTimeInterval fromLog = [self secondsFromEvent:[self tailJSONOf:sp]];
    NSTimeInterval hb = [self lastActivityTime];
    check(hb > 0 && hb >= fromLog,
          @"心跳完全取自工具日志，未掺入 changes-index",
          [NSString stringWithFormat:@"心跳 %@", [self agoText:hb]]);
    if (idxM > hb + 5)
        printf("      注：changes-index 比心跳新 %.0f 秒（正是不会误报的证明）\n", idxM - hb);

    printf("== 3. 最近动作解析 ==\n");
    NSString *act = [self readLastAction];
    check(act.length > 0, @"能解析出最近动作", [self truncate:act max:40]);

    printf("== 4. 忙闲判定（主判据：回合在途；兜底：心跳 %.0f 秒）==\n", kBusyWindow);
    NSTimeInterval now = [[NSDate date] timeIntervalSince1970];
    #define T_BUSY(h, fl) [self isBusyOnline:YES heart:(h) inFlight:(fl) inFlightSince:((fl) ? now - 5 : 0) hasTranscript:YES now:now window:kBusyWindow]
    #define N_BUSY(h)     [self isBusyOnline:YES heart:(h) inFlight:NO inFlightSince:0 hasTranscript:NO now:now window:kBusyWindow]

    check(T_BUSY(now - 900, YES), @"回合在途（深度思考/工具执行）→ 忙，日志再旧也绿", nil);
    check(!T_BUSY(now, NO),       @"回合已交付 → 立刻红（哪怕 1 秒前刚调过工具）", nil);
    check(!T_BUSY(now - 900, NO), @"回合已交付 → 红", nil);
    check(!T_BUSY(0, NO),         @"已交付且无任何活动 → 红", nil);
    check(![self isBusyOnline:NO heart:now inFlight:YES inFlightSince:now hasTranscript:YES now:now window:kBusyWindow],
          @"离线 → 红（即使在途）", nil);
    check(![self isBusyOnline:YES heart:now - 900 inFlight:YES inFlightSince:now - kMaxInFlight - 10 hasTranscript:YES now:now window:kBusyWindow],
          [NSString stringWithFormat:@"在途超过 %.0f 分钟（挂死保护）→ 红", kMaxInFlight / 60], nil);
    // 兜底路径：读不到 transcript 时才启用心跳窗口
    check(N_BUSY(now),        @"无 transcript · 心跳新鲜 → 忙", nil);
    check(!N_BUSY(now - kBusyWindow - 1),
          [NSString stringWithFormat:@"无 transcript · 心跳超 %.0f 秒 → 闲", kBusyWindow], nil);
    check(!N_BUSY(0),         @"无 transcript · 无心跳 → 闲", nil);
    // V2.5：快照 working 条目也要讲时效（修「没活儿灯还一直闪」）
    check([self workingItemCounts:@{@"state":@"working",@"kind":@"task",
                                    @"lastActivityAt":@( (now - 60) * 1000 )}
                snapMtime:0 now:now],
          @"快照 working · 1 分钟前有动静 → 忙（不灭灯纪律不变）", nil);
    check(![self workingItemCounts:@{@"state":@"working",@"kind":@"task",
                                     @"lastActivityAt":@( (now - kMaxInFlight - 60) * 1000 )}
                snapMtime:0 now:now],
          [NSString stringWithFormat:@"快照 working · 超过 %.0f 分钟没动静 → 放过（异常中断残留自愈）", kMaxInFlight / 60], nil);
    check([self workingItemCounts:@{@"state":@"working",@"kind":@"task"}
                snapMtime:now - 30 now:now],
          @"快照 working · 无 lastActivityAt · 快照新鲜 → 忙（mtime 兜底）", nil);
    check(![self workingItemCounts:@{@"state":@"working",@"kind":@"task"}
                snapMtime:now - kMaxInFlight - 30 now:now],
          @"快照 working · 无 lastActivityAt · 快照也旧 → 闲", nil);
    check(![self workingItemCounts:@{@"state":@"idle",@"kind":@"task",
                                     @"lastActivityAt":@( now * 1000 )}
                snapMtime:0 now:now],
          @"快照 idle → 闲（时效不改变原判定）", nil);
    #undef T_BUSY
    #undef N_BUSY

    printf("== 5. transcript 回合状态（V1.2 主信号）==\n");
    NSString *tp = [self latestTranscript];
    check(tp.length > 0, @"能找到最新会话 transcript",
          tp ? [NSString stringWithFormat:@"…%@/%@",
                [[tp stringByDeletingLastPathComponent] lastPathComponent],
                tp.lastPathComponent] : @"未找到");
    NSArray<NSDictionary *> *recent = tp ? [self tailEventsOf:tp bytes:kTailBytes limit:60] : @[];
    check(recent.count > 0, @"尾部事件可批量解析",
          [NSString stringWithFormat:@"%lu 条", (unsigned long)recent.count]);
    NSDictionary *ev2 = recent.firstObject;
    NSTimeInterval evT = [self secondsFromEvent:ev2];
    check(evT > 0, @"事件行带可用 timestamp", [self agoText:evT]);
    check([self eventName:ev2].length > 0, @"事件可翻译成中文动作名", [self eventName:ev2]);

    // 实测结构校验：role / status 在顶层 —— 这是 V1.2 踩过的坑，必须回归验证。
    // 注意：不能只看尾部若干行（连续调几十个工具时尾部可能一条 message 都没有），
    // 必须扫全文件，断言才不随运行时刻漂移。
    NSString *full = [NSString stringWithContentsOfFile:tp encoding:NSUTF8StringEncoding error:nil];
    NSDictionary *lastMsg = nil;
    NSInteger nBookkeeping = 0, nReal = 0;
    for (NSString *ln in [full componentsSeparatedByString:@"\n"]) {
        NSString *t = [ln stringByTrimmingCharactersInSet:
                           [NSCharacterSet whitespaceAndNewlineCharacterSet]];
        if (t.length < 2) continue;
        NSDictionary *e = [NSJSONSerialization JSONObjectWithData:[t dataUsingEncoding:NSUTF8StringEncoding]
                                                         options:0 error:nil];
        if (![e isKindOfClass:[NSDictionary class]] || !e.count) continue;
        if ([[self strField:e key:@"type"] isEqualToString:@"message"]) lastMsg = e;
        if ([self isSubstantiveEvent:e]) nReal++; else nBookkeeping++;
    }
    check(lastMsg != nil, @"能扫到 message 行（全文件）",
          [NSString stringWithFormat:@"%ld 行", (long)[full componentsSeparatedByString:@"\n"].count]);
    check([self strField:lastMsg key:@"role"].length > 0,
          @"message 行的 role 在顶层、能取到（V1.2 修正点）",
          [self strField:lastMsg key:@"role"]);

    // 记账类事件必须被跳过，否则会把"已交付"顶掉造成假绿
    check(![self isSubstantiveEvent:@{@"type": @"file-history-snapshot"}],
          @"文件快照不算实质事件（防假绿）", nil);
    check(![self isSubstantiveEvent:@{@"type": @"ai-title"}], @"标题整理不算实质事件", nil);
    check([self isSubstantiveEvent:@{@"type": @"reasoning"}], @"reasoning 是实质事件", nil);
    check([self isSubstantiveEvent:@{@"type": @"message", @"role": @"assistant"}], @"message 是实质事件", nil);
    check(nBookkeeping > 0, @"真实日志里确实存在记账类事件（证明跳过逻辑必要）",
          [NSString stringWithFormat:@"全文件 实质 %ld · 记账 %ld 行", (long)nReal, (long)nBookkeeping]);

    NSDictionary *st = [self transcriptState];
    check(st.count > 0, @"transcriptState 组装完整",
          st ? [NSString stringWithFormat:@"在途=%@ · %@ · %@", st[@"inFlight"], st[@"name"],
                [self agoText:[st[@"time"] doubleValue]]] : @"nil");

    // eventInFlight 纯函数：六种末行形态（字段一律在顶层）
    check([self eventInFlight:@{@"type": @"reasoning"}], @"末行=reasoning → 在途（深度思考）", nil);
    check([self eventInFlight:@{@"type": @"function_call"}], @"末行=function_call → 在途（工具执行中）", nil);
    check([self eventInFlight:@{@"type": @"function_call_result"}], @"末行=function_call_result → 在途（想下一步）", nil);
    check([self eventInFlight:@{@"type": @"message", @"role": @"user"}], @"末行=user → 在途（刚接单）", nil);
    check(![self eventInFlight:@{@"type": @"message", @"role": @"assistant", @"status": @"completed"}],
          @"末行=assistant completed → 已交付（红）", nil);
    check([self eventInFlight:@{@"type": @"message", @"role": @"assistant", @"status": @"incomplete"}],
          @"末行=assistant incomplete → 在途", nil);
    check([self eventInFlight:@{@"type": @"message", @"role": @"assistant"}],
          @"末行=assistant 无 status → 视为在途", nil);
    // 打断/中止：按 ESC 或编译 aborted —— 回合其实已结束，必须转红
    check(![self eventInFlight:@{@"type": @"message", @"role": @"assistant", @"status": @"incomplete",
                                 @"content": @[@{@"type": @"output_text", @"text": @"Interrupted by user"}]}],
          @"末行=打断(Interrupted) → 已结束（红）", nil);
    check(![self eventInFlight:@{@"type": @"message", @"role": @"assistant", @"status": @"incomplete",
                                 @"content": @[@{@"type": @"output_text", @"text": @"aborted"}]}],
          @"末行=aborted → 已结束（红）", nil);
    check([[self eventName:@{@"type": @"message", @"role": @"assistant", @"status": @"incomplete",
                             @"content": @[@{@"text": @"Interrupted by user"}]}] isEqualToString:@"已被打断"],
          @"打断的文案不写成「已交付回复」", nil);
    check([[self eventName:@{@"type": @"message", @"role": @"assistant", @"status": @"incomplete",
                             @"content": @[@{@"text": @"正在写的半截话"}]}] isEqualToString:@"正在回复"],
          @"普通中间态文案为「正在回复」", nil);

    printf("== 6. 文案处理 ==\n");
    check([[self oneLine:@"ls -la\nhead -5"] isEqualToString:@"ls -la head -5"], @"多行命令压成一行", nil);
    check([self truncate:@"0123456789" max:5].length == 6, @"超长截断并加省略号", [self truncate:@"0123456789" max:5]);
    check([[self truncate:@"abc" max:5] isEqualToString:@"abc"], @"未超长不改动", nil);

    printf("== 7. 调用计数缓存 ==\n");
    NSInteger c1 = [self todayCallCount];
    NSInteger c2 = [self todayCallCount];
    check(c1 == c2 && c1 > 0, @"今日调用数可统计且缓存一致",
          [NSString stringWithFormat:@"%ld 次", (long)c1]);

    printf("== 7.5 今日消耗（V1.9 已用累计差值）==\n");
    {   // 新分支会写基线文件：先存原文，测完还原，别污染真实基线
        NSString *store = self.spentStorePath;
        NSString *orig = [NSString stringWithContentsOfFile:store encoding:NSUTF8StringEncoding error:nil];
        check([[Delegate formatSpent:0] isEqualToString:@"0"], @"0 记 0", nil);
        check([[Delegate formatSpent:0.03] isEqualToString:@"0"], @"微量四舍五入为 0", nil);
        check([[Delegate formatSpent:12.5] isEqualToString:@"12.5"], @"一位小数", nil);
        check([[Delegate formatSpent:123.04] isEqualToString:@"123"], @"整数不带小数尾", nil);
        self.spentDayKey = [self todayKey];
        self.spentBaselineValid = YES;
        self.spentBaseline = 100.0;
        self.usedBaseline = 1000.0;
        self.lastSpentCalc = 0;
        [self updateSpentWithRaw:90.0 used:1010.0];
        check(self.todaySpent == 10.0, @"消耗 = 已用累计 − 已用基线", [Delegate formatSpent:self.todaySpent]);
        [self updateSpentWithRaw:95.0 used:1020.0];
        check(self.todaySpent == 10.0, @"10 分钟节流内不重算", nil);
        self.lastSpentCalc = 0;
        [self updateSpentWithRaw:110.0 used:1015.0];   // 签到发分把余额抬高（90→110），但已用只涨了 5
        check(self.todaySpent == 15.0, @"发分不再吞掉消耗（V1.9 关键回归）", [Delegate formatSpent:self.todaySpent]);
        self.lastSpentCalc = 0;
        [self updateSpentWithRaw:120.0 used:1000.0];   // 已用回落（周期翻转）→ 记 0 并重置锚点
        check(self.todaySpent == 0.0 && self.usedBaseline == 1000.0, @"已用回落记 0 并重置锚点", nil);
        self.spentBaselineValid = NO;
        if (orig) [orig writeToFile:store atomically:YES encoding:NSUTF8StringEncoding error:nil];
        else [[NSFileManager defaultManager] removeItemAtPath:store error:nil];
    }

    printf("== 7.6 当前任务=会话 ai-title（V1.5 三修）==\n");
    NSString *tmp = [NSTemporaryDirectory() stringByAppendingPathComponent:@"wblight_title_test.jsonl"];
    NSMutableString *big = [NSMutableString string];
    for (int k = 0; k < 40; k++) [big appendFormat:@"{\"type\":\"function_call\",\"n\":%d,\"pad\":\"%@\"}\n",
                                 k, [@"" stringByPaddingToLength:200 withString:@"x" startingAtIndex:0]];
    [big appendFormat:@"{\"type\":\"ai-title\",\"aiTitle\":\"旧标题\"}\n"];
    [big appendFormat:@"%@\n", [@"" stringByPaddingToLength:300 withString:@"y" startingAtIndex:0]];
    [big appendString:@"{\"type\":\"ai-title\",\"aiTitle\":\"一些小应用的开发\"}\n"];
    [big writeToFile:tmp atomically:YES encoding:NSUTF8StringEncoding error:nil];
    check([[self latestTitleInFile:tmp] isEqualToString:@"一些小应用的开发"],
          @"多块扫描取最后一个 ai-title", [self latestTitleInFile:tmp]);
    [@"没有标题的文件\n" writeToFile:tmp atomically:YES encoding:NSUTF8StringEncoding error:nil];
    check([self latestTitleInFile:tmp] == nil, @"无 ai-title 返回 nil", nil);
    [[NSFileManager defaultManager] removeItemAtPath:tmp error:nil];

    printf("== 8. 当前真实状态 ==\n");
    NSDictionary *live = [self transcriptState];
    self.online = [self workBuddyRunning];
    self.lastActivity = [self lastActivityTime];
    self.heartTime = MAX(live ? [live[@"time"] doubleValue] : 0, self.lastActivity);
    self.inFlight = live ? [live[@"inFlight"] boolValue] : NO;
    self.busy = [self isBusyOnline:self.online heart:self.heartTime
                           inFlight:self.inFlight
                     inFlightSince:(live ? [live[@"time"] doubleValue] : 0)
                     hasTranscript:(live != nil)
                               now:now window:kBusyWindow];
    NSString *evName = [live[@"name"] isKindOfClass:[NSString class]] ? live[@"name"] : self.lastAction;
    printf("      在线=%s  忙=%s  回合在途=%s  末事件=%s\n",
           self.online ? "是" : "否", self.busy ? "是" : "否",
           self.inFlight ? "是" : "否",
           (evName ?: @"—").UTF8String);
    [self refreshCurrentTask];   // 实测任务抽取（512KB 窗口）
    printf("      最近活动=%s  今日调用=%ld  当前任务=%s\n",
           [self agoText:self.lastActivity].UTF8String, (long)[self todayCallCount],
           (self.currentTask ?: @"—").UTF8String);
    printf("      灯色=%s\n", self.busy ? "绿色闪烁 W（干活中）" : "红色常亮 W（闲置）");
    check(self.online, @"WorkBuddy 进程在线（bundle id 主路径）", nil);
    check([self workBuddyRunning], @"在线检测可用（含按进程名兜底，防对方 bundle id 不同）", nil);
    NSURL *wbu = [self workBuddyAppURL];
    check(wbu != nil, @"能定位 WorkBuddy 的 app 位置（菜单「打开 WorkBuddy」用）",
          wbu.path);

    printf("\n== 7.75 UTF-8 分块兜底（V2.2 修乱码）==\n");
    {   // 块边界切在多字节字符中间：旧算法整块按 Latin-1 硬解 → 全行乱码；新算法丢残字节重解
        NSString *full = @"当前任务：一些小应用的开发";
        NSData *fd = [full dataUsingEncoding:NSUTF8StringEncoding];
        NSData *cut = [fd subdataWithRange:NSMakeRange(2, fd.length - 2)];   // 故意切进"当"字中间
        NSString *got = [self utf8FromChunk:cut];
        check(got != nil && [got containsString:@"任务"] && [got containsString:@"小应用"],
              @"块头切在汉字中间也能正确解码（不再 Latin-1 硬解）",
              [got substringToIndex:MIN(12, got.length)] ?: @"nil");
    }

    printf("\n== 7.7 提示音（V1.8，移植自 Codex小灯）==\n");
    NSArray<NSDictionary<NSString *, NSString *> *> *sounds = [self availableSounds];
    check(sounds.count >= 10, @"能列出系统提示音（≥10 个音源）",
          [NSString stringWithFormat:@"%lu 个，默认=%@",
           (unsigned long)sounds.count, [self defaultSoundPath].pathExtension ?: @""]);
    check([[NSFileManager defaultManager] fileExistsAtPath:[self defaultSoundPath]],
          @"默认提示音文件存在（Tink.aiff）", [self defaultSoundPath]);
    {   // 偏好读写闭环：写 → 落盘（直接读 plist 文件取证）→ 走一遍真路径 → 原样还原
        // V2.3 修两件事：
        // ① 以前这里没有 synchronize，新实例读到的是磁盘上的旧值，用例时好时坏
        //   （以前之所以 PASS，是磁盘里恰好留着上一轮的 /tmp/x.aiff，等于自己骗自己）。
        // ② 还原时会把「键本来不存在」写成 SoundEnabled=false / SoundPath=""，
        //   于是默认「提示音开 + Tink」被压成「关」，等于自测把用户设置搞坏了。
        //   现在：键原来不存在就 removeObjectForKey，绝不凭空造出一个 false。
        NSUserDefaults *d = [self soundPrefs];
        id oldEnObj = [d objectForKey:@"SoundEnabled"];
        id oldPathObj = [d objectForKey:@"SoundPath"];
        [d setBool:NO forKey:@"SoundEnabled"];
        [d setObject:@"/tmp/x.aiff" forKey:@"SoundPath"];
        [d synchronize];
        // 取证方式：**另起一个进程**去读（等价于"重启 app 后再看还在不在"）。
        // 为什么不直接读 plist 文件：cfprefsd 落盘是异步的，刚 synchronize 完文件里可能还没有，
        // 直接读文件会假失败；而 cfprefsd 本身是权威缓存，新进程读到的就是重启后能读到的。
        NSString *selfPath = NSProcessInfo.processInfo.arguments.firstObject ?: @"";
        if (!selfPath.length || ![selfPath hasPrefix:@"/"]) {
            selfPath = [NSFileManager.defaultManager.currentDirectoryPath stringByAppendingPathComponent:selfPath];
        }
        NSString *childOut = @"";
        @try {
            NSTask *t = [NSTask new];
            t.executableURL = [NSURL fileURLWithPath:selfPath];
            t.arguments = @[@"--soundpref"];
            NSPipe *pipe = [NSPipe pipe];
            t.standardOutput = pipe;
            [t launchAndReturnError:nil];
            NSData *data = [pipe.fileHandleForReading readDataToEndOfFileAndReturnError:nil];
            [t waitUntilExit];
            childOut = [[[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding]
                        stringByTrimmingCharactersInSet:[NSCharacterSet whitespaceAndNewlineCharacterSet]] ?: @"";
        } @catch (NSException *e) { childOut = @""; }
        Delegate *probe = [Delegate new];
        [probe loadSoundPrefs];
        BOOL ok = [childOut isEqualToString:@"0|/tmp/x.aiff"] &&
                  (probe.soundEnabled == NO) && [probe.soundPath isEqualToString:@"/tmp/x.aiff"];
        check(ok, @"提示音偏好读写闭环（另起进程读回，= 重启后还在）",
              [NSString stringWithFormat:@"子进程=%@", childOut.length ? childOut : @"(没读到)"]);
        if (oldEnObj) { [d setBool:[oldEnObj boolValue] forKey:@"SoundEnabled"]; } else { [d removeObjectForKey:@"SoundEnabled"]; }
        if (oldPathObj) { [d setObject:oldPathObj forKey:@"SoundPath"]; } else { [d removeObjectForKey:@"SoundPath"]; }
        [d synchronize];
    }

    printf("\n结果：%s（失败 %d 项）\n", fails == 0 ? "全部通过 ✅" : "存在失败 ❌", fails);
    return fails;
}

@end

int main(int argc, const char *argv[]) {
    @autoreleasepool {
        if (argc > 1 && strcmp(argv[1], "--selftest") == 0) {
            Delegate *d = [Delegate new];
            return [d runSelfTest];
        }
        if (argc > 1 && strcmp(argv[1], "--replay") == 0) {
            Delegate *d = [Delegate new];
            return [d replayTranscript];
        }
        if (argc > 1 && strcmp(argv[1], "--state") == 0) {
            Delegate *d = [Delegate new];
            printf("%s\n", [d stateLine].UTF8String);
            return 0;
        }
        // V2.3：给自测用的探针——另起一个进程把提示音偏好原样打印出来（格式 开关|路径）
        if (argc > 1 && strcmp(argv[1], "--soundpref") == 0) {
            NSUserDefaults *d = [NSUserDefaults standardUserDefaults];
            id en = [d objectForKey:@"SoundEnabled"];
            NSString *pa = [d stringForKey:@"SoundPath"] ?: @"";
            printf("%d|%s\n", en ? [en boolValue] : 1, pa.UTF8String);
            return 0;
        }
        NSApplication *app = [NSApplication sharedApplication];
        [app setActivationPolicy:NSApplicationActivationPolicyAccessory];
        Delegate *d = [Delegate new];
        app.delegate = d;
        [app run];
    }
    return 0;
}

```

---

## 4. 编译 / 使用 / 调参

### 编译并启动

```bash
cd ~/Documents/Work/大AI开发/WB小灯     # 或你自己放源码的目录
clang -fobjc-arc -framework Cocoa -o WBLight WBLight.m
./WBLight --selftest                    # 先看 46 项是否全 PASS
./WBLight --state                       # 打一行当前状态（调试/采样用）
./WBLight --replay                      # 拿真实 transcript 全量重放，验证判定逻辑
```

想直接编一个能给别人用的**通用二进制**（Intel + Apple Silicon 都能跑）：

```bash
clang -fobjc-arc -framework Cocoa -arch arm64 -arch x86_64 -o WBLight WBLight.m
lipo -info WBLight                      # 应输出 x86_64 arm64
```

### 打成 App（双击即用）

```bash
mkdir -p "WB小灯.app/Contents/MacOS"
cp WBLight "WB小灯.app/Contents/MacOS/WBLight"
chmod +x "WB小灯.app/Contents/MacOS/WBLight"
cat > "WB小灯.app/Contents/Info.plist" <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>CFBundleExecutable</key><string>WBLight</string>
    <key>CFBundleIdentifier</key><string>io.wblight.menubar</string>
    <key>CFBundleName</key><string>WB小灯</string>
    <key>CFBundleDisplayName</key><string>WB小灯</string>
    <key>CFBundlePackageType</key><string>APPL</string>
    <key>CFBundleShortVersionString</key><string>1.2</string>
    <key>CFBundleVersion</key><string>3</string>
    <key>LSUIElement</key><true/>
    <key>NSHighResolutionCapable</key><true/>
    <key>LSMinimumSystemVersion</key><string>12.0</string>
</dict>
</plist>
EOF
codesign --force --deep --sign - "WB小灯.app"
xattr -cr "WB小灯.app"
open "WB小灯.app"
```

### 发给别人用（交付前必看，实测踩过）

App 是 ad-hoc 自签名（`codesign -s -`），没有 Apple 开发者账号就没法公证。
别人从微信/浏览器/邮件收到后，文件会被打上 `com.apple.quarantine` 隔离标记 →
**双击一定被拦**（实测 macOS 26.5：「无法打开，因为 Apple 无法检查…」）。

**最关键的一条（实测可复现，顺序反了修不好）：**

> **必须先删隔离标记，再双击。**
> 一旦双击被拦过，bundle 就被系统锁上，`xattr` 删除会报 `Operation not permitted`。

对照实验（同一台机器，同样从 zip 解压，走 `ditto` 原生解压）：

| 组 | 操作顺序 | 结果 |
|----|---------|------|
| 甲 | 先双击（被拦）→ 再 `xattr -d com.apple.quarantine` | ❌ `Operation not permitted`，仍打不开 |
| 丙 | 先双击（被拦）→ 再 `xattr -cr` | ❌ 对 `Contents/_CodeSignature/*` 报 EPERM，仍打不开 |
| 乙 | 先 `xattr -d com.apple.quarantine` → 双击 | ✅ 打开 |
| 丁 | 先 `xattr -cr` → 双击 | ✅ 打开 |

被拦过之后的补救（实测均有效）：**删掉这份、从 zip 重新解压一份，再走「先删标记」的流程**；
或走 系统设置 → 隐私与安全性 → 安全性的「仍要打开」。

交付给别人的正确姿势（这一段直接抄给使用者）：

```bash
# 1) 解压后，先别双击！开终端
# 2) 敲命令，注意 cr 后面有空格，然后把 .app 拖进终端窗口补全路径
xattr -cr /Applications/WB小灯.app
# 3) 现在才双击
open /Applications/WB小灯.app
```

顺带结论：
- `xattr -d com.apple.quarantine <app>`（只删外层）实测也够用，隔离标记在 bundle 的 7 个条目上都有，但 Gatekeeper 只看最外层那个
- 隔离标记是**解压时从 zip 传染**下来的：给 zip 打标记 → `ditto` 解压 → 里面的 .app 也带标记
- 想让对方零操作，只剩两条路：**买开发者账号做公证**（付费），或者**让对方自己编译**（编译产物不带隔离标记，永远不弹窗）

### 开机自启（推荐，省得每次手动开）

系统设置 → 通用 → 登录项 → 点 `+` → 选中 `WB小灯.app` → 添加。

### 可调参数（源码顶部，改完重新编译即可）

| 参数 | 默认 | 作用 |
|------|------|------|
| `kBusyWindow` | **20 秒** | **兜底**心跳窗口。只在读不到 transcript 时才生效。调大只会让"无 transcript"场景多绿一会儿 |
| `kMaxInFlight` | **900 秒**（15 分） | 回合在途的最长时效。防 agent 挂死导致永远绿 |
| `kPollInterval` | 1.0 秒 | 状态巡检间隔，也是"交付→转红"的最长延迟。想更跟手可调 0.5 |
| `kBlinkInterval` | 0.55 秒 | 绿灯闪烁半周期。想快闪调 0.3 |
| `kTailBytes` | 65536 | 每次只读文件尾部这么多字节，避免扫大文件 |
| 绿色/红色 | 见 `render` | `colorWithCalibratedRed:green:blue:` 数值，随便调 |

### 换判定源（如果 WorkBuddy 改了日志格式）

两处：

1. **主判据**：`latestTranscript`（去哪找会话文件）+ `eventName` / `eventInFlight` / `isSubstantiveEvent`（事件怎么解读）
2. **兜底**：`lastActivityTime`（审计日志路径与时间戳字段），以及 `spoolDir` / `todayArchiveFile`

自测第 1、5 节会立刻告诉你两个信号源是否还读得到。拿不准则先用 `--replay` 看真实数据长什么样。

---

## 5. 怎么自证它没误报

### 5.1 重放验证（最硬的一条）

```bash
./WBLight --replay
```

它会**读你真实的整个 transcript（上千行）**，用同一套判定逻辑逐行重算，输出统计和最近 16 次灯色翻转。实测输出（刚哥本机，2017 行）：

```
重放文件：/Users/<你的用户名>/.workbuddy/projects/Users-<你的用户名>-WorkBuddy-2026-09-10-15-58-39/630a11c1-60f4-453e-9914-dfb429466a7e.jsonl

总行数 2021
  实质事件 绿=1491 红=272   （记账类跳过 258 行）
  深度思考行 355 · 交付事件 270 · 状态翻转 544 次

最近 16 次灯色翻转（真实数据）：
  绿 → 红   （已交付回复）
  红 → 绿   （调用工具）
  绿 → 红   （已交付回复）
  红 → 绿   （调用工具）
  绿 → 红   （已交付回复）
  红 → 绿   （调用工具）
  绿 → 红   （已交付回复）
  红 → 绿   （调用工具）
  绿 → 红   （已交付回复）
  红 → 绿   （调用工具）
  绿 → 红   （已交付回复）
  红 → 绿   （调用工具）
  绿 → 红   （已交付回复）
  红 → 绿   （调用工具）
  绿 → 红   （已交付回复）
  红 → 绿   （调用工具）

结论：「已交付」全部正确判红，无假绿 ✅
```

翻转曲线干净地呈现「调工具 → 绿 / 交付 → 红」，零假绿。

### 5.2 一行状态采样

```bash
./WBLight --state        # 17:47:45   🟢绿  在线=是 在途=是  末事件=调用工具
```

想连续观察就 `for i in $(seq 1 20); do ./WBLight --state; sleep 1; done`。

### 5.3 直接看原始数据

```bash
# 当前会话 transcript 的最后几行长什么样：
ls -t ~/.workbuddy/projects/*/*.jsonl | head -1 | xargs tail -3
```

重点看末行的 `type` / `role` / `status` —— 对照第 2 节那张对照表，自己就能算出灯该是什么颜色。

---

## 6. 自测（本机实测，46 项 PASS）

```
== 1. 心跳源：只认工具调用日志 ==
PASS  spool 有日志文件  (audit-spool-73160-2026-09-11.jsonl)
PASS  tailJSONOf 能解析出最后一行事件  (command-safety.sandbox-executed)
PASS  事件里有可用 timestamp  (21 秒前)
PASS  合并心跳 > 0  (21 秒前)
PASS  当天归档路径可构造  (2026-09-11.jsonl)
== 2. 误报源剔除（V1.1 修正）==
PASS  心跳完全取自工具日志，未掺入 changes-index  (心跳 21 秒前)
      注：changes-index 比心跳新 16 秒（正是不会误报的证明）
== 3. 最近动作解析 ==
PASS  能解析出最近动作  (Grep "/Users/<你的用户名>/Documents/Work/大AI开…)
== 4. 忙闲判定（主判据：回合在途；兜底：心跳 20 秒）==
PASS  回合在途（深度思考/工具执行）→ 忙，日志再旧也绿
PASS  回合已交付 → 立刻红（哪怕 1 秒前刚调过工具）
PASS  回合已交付 → 红
PASS  已交付且无任何活动 → 红
PASS  离线 → 红（即使在途）
PASS  在途超过 15 分钟（挂死保护）→ 红
PASS  无 transcript · 心跳新鲜 → 忙
PASS  无 transcript · 心跳超 20 秒 → 闲
PASS  无 transcript · 无心跳 → 闲
== 5. transcript 回合状态（V1.2 主信号）==
PASS  能找到最新会话 transcript  (…Users-<你的用户名>-WorkBuddy-2026-09-10-15-58-39/630a11c1-60f4-453e-9914-dfb429466a7e.jsonl)
PASS  尾部事件可批量解析  (19 条)
PASS  事件行带可用 timestamp  (2 秒前)
PASS  事件可翻译成中文动作名  (调用工具)
PASS  能扫到 message 行（全文件）  (2212 行)
PASS  message 行的 role 在顶层、能取到（V1.2 修正点）  (assistant)
PASS  文件快照不算实质事件（防假绿）
PASS  标题整理不算实质事件
PASS  reasoning 是实质事件
PASS  message 是实质事件
PASS  真实日志里确实存在记账类事件（证明跳过逻辑必要）  (全文件 实质 1937 · 记账 274 行)
PASS  transcriptState 组装完整  (在途=1 · 调用工具 · 2 秒前)
PASS  末行=reasoning → 在途（深度思考）
PASS  末行=function_call → 在途（工具执行中）
PASS  末行=function_call_result → 在途（想下一步）
PASS  末行=user → 在途（刚接单）
PASS  末行=assistant completed → 已交付（红）
PASS  末行=assistant incomplete → 在途
PASS  末行=assistant 无 status → 视为在途
PASS  末行=打断(Interrupted) → 已结束（红）
PASS  末行=aborted → 已结束（红）
PASS  打断的文案不写成「已交付回复」
PASS  普通中间态文案为「正在回复」
== 6. 文案处理 ==
PASS  多行命令压成一行
PASS  超长截断并加省略号  (01234…)
PASS  未超长不改动
== 7. 调用计数缓存 ==
PASS  今日调用数可统计且缓存一致  (259 次)
== 8. 当前真实状态 ==
      在线=是  忙=是  回合在途=是  末事件=调用工具
      最近活动=22 秒前  今日调用=259
      灯色=绿色闪烁 W（干活中）
PASS  WorkBuddy 进程在线（bundle id 主路径）
PASS  在线检测可用（含按进程名兜底，防对方 bundle id 不同）
PASS  能定位 WorkBuddy 的 app 位置（菜单「打开 WorkBuddy」用）  (/Applications/WorkBuddy.app)

结果：全部通过 ✅（失败 0 项）
```

覆盖范围：心跳源可读性（5）、误报源剔除（1）、最近动作解析（1）、
忙闲判定主判据 + 兜底（9）、**transcript 回合判定（19，含 role/status 层级回归、
记账事件跳过、打断识别）**、文案处理（3）、调用计数缓存（1）、当前真实状态（3，含
在线检测兜底 / app 定位）。

所有断言都扫全文件，不依赖运行时刻 —— 连续跑多遍结果稳定，不漂移。

---

## 7. 踩坑记录

| # | 坑 | 现象 | 解法 |
|---|----|------|------|
| 1 | **changes-index 被当心跳** | 干完活灯不红，一发消息就变绿 | V1.1 剔除。心跳只认工具调用日志的 `timestamp` |
| 2 | **sessions 的 lastHeartbeat 被当心跳** | 没干活也一直绿 | 它是固定 30 秒一次的「存活」心跳，与干活无关，剔除 |
| 3 | **audit-log 目录 mtime 被当心跳** | 归档/轮转时假动作 | 只读日志**行内** timestamp，不看目录时间 |
| 4 | 菜单栏文字被系统按"禁用态"压灰 | 信息行发暗 | 信息行不用普通 NSMenuItem 文字，改用 `NSMenuItem.view` 自定义 NSView + NSTextField |
| 5 | 闪烁如果切换"有字/无字"会让图标宽度跳动 | 菜单栏抖动 | 改为切换 **alpha**（100% ↔ 22%），宽度稳定、观感是"忽明忽暗" |
| 6 | 忙 → 闲时的闪烁残留 | 停在暗态像"没显示" | 转闲时立刻 `stopBlink` 并把 alpha 复位 |
| 7 | 日志文件越来越大（单日 150KB） | 每秒全量读浪费 | 只 seek 到尾部读 64KB；末行若是半截写入会自动往前找 |
| 8 | 大文件每秒重扫做调用计数 | CPU 白耗 | 按「文件 size + mtime」做缓存 key，没变化就不重算 |
| 9 | 日志是异步批量落盘 | 心跳比动作晚几秒~几十秒 | 已知边界，只影响"多绿一会儿"，不会误绿 |
| 10 | **拿 CPU 当"在干活"的信号** | 想用 `proc_pid_rusage` 读进程 CPU 判断是否在生成 | **路线死亡**。实测 19 个 WorkBuddy 进程在深度思考时 CPU 增量全部 < 0.01s/s —— 推理跑在云端，本地只有流式渲染。别再试 |
| 11 | **`role` / `status` 读错了层级** | 交付后绿灯**永远不灭**（因为 completed 判不出来，一直算"在途"） | V1.2 最大的坑。抓 2017 行真实日志才定位：**它们是顶层字段**，不在 `message` 子字典里。写成 `e[@"message"][@"role"]` 永远取到空字符串 |
| 12 | **记账事件插花落在末行** | 保存文件备份（`file-history-snapshot`）、生成会话标题（`ai-title`）会夹在工具调用前后。若恰是末行，会把"已交付"顶掉 → 假绿 | 新增 `isSubstantiveEvent`，判定时从后往前跳过记账事件，取最近的实质事件。实测真实日志里 243 行属于此类 |
| 13 | **按 ESC 打断被算成"在途"** | 打断写入的是 `assistant + incomplete` + 内容 `Interrupted by user`。按"incomplete 即在途"会一路绿到 15 分钟挂死保护 | 识别 `Interrupted by user` / `aborted` → 回合已结束 → 立刻红 |
| 14 | **"兜底"写成了"并联"** | 主判据之外又并联一个 20 秒心跳窗口，干完活后照样多绿 20 秒 | 兜底必须是**真兜底**：只有读不到 transcript 时才启用 |
| 15 | 自测断言依赖运行时刻 | "尾部能找到 message 行"这条，在连续调几十个工具时尾部真的一条 message 都没有 → 偶发 FAIL | 改成扫全文件取最后一条 message，断言不再随运行状态漂移 |
| 16 | 宏参数名和选择器关键字同名 | `#define BUSY(heart, fl)` 里 `heart:heart` 的**关键字也被预处理器替换掉**了，编译报"找不到选择器"且报错信息极具误导性 | 宏参数改名 `h` / `fl`。见源码 `runSelfTest` 第 4 节 |
| 17 | **发给别人的 app 被 Gatekeeper 拦** | 别人双击弹「无法打开，因为 Apple 无法检查…」。adhoc 签名、无公证，必然如此 | 交付时随包附「先 `xattr -cr` 再双击」的说明。**顺序不能反**：先双击被拦过就删不掉了，得重新解压一份。详见第 4 节 |
| 18 | **在线检测写死 bundle id** | 对方装的是 bundle id 不同的变体（分发渠道不同会有），会「永远是红灯」，看着像坏了 | 主路径仍按 bundle id，兜底再按运行进程名含 `WorkBuddy` 找一遍；`openWB:` 同样加了兜底 |

---

## 7.1 V1.2 排查路线（留个记录，下次别再走弯路）

刚哥反馈：「深度思考时灯还是红的，而且干完活绿灯不灭」。排查顺序：

1. **先怀疑心跳太旧** → 把窗口从 60 秒收紧 → **没用**（因为思考时压根没有审计日志写入）
2. **试 CPU 信号** → 读 `proc_pid_rusage` 拿 19 个 WorkBuddy 进程的 CPU 时间 → **全部静默**，增量 < 0.01s/s → **此路不通，推理在云端**
3. **翻本机所有会写文件的信号源** → 试了 `changes-index`（打字就写，噪声）、`sessions/*.json` 的 `lastHeartbeat`（固定 30 秒）、`poller-lease`（5 秒一跳）、`claw-state/weixin`（机器人游标）→ **全部与忙闲无关**
4. **找到真信号** → `projects/<会话>.jsonl`（会话事件日志），含 `reasoning` / `function_call` / `message`，**实时落盘、逐事件追加**
5. **按"回合在途"重写判定** → 但一开始 `role` 读错层级，绿灯永不灭
6. **拿 2017 行真实日志逐条 dump** → 发现 `role` / `status` 在顶层；同时发现 `file-history-snapshot` / `ai-title` 是插花的记账事件、`Interrupted by user` 是打断态
7. **加 `--replay` 重放模式** → 用真实数据全量验证：269 次交付事件 100% 正确判红，零假绿

**教训**：不要凭"合理猜测"去读别人写的 JSON —— 一定要 `dump` 原始结构确认字段层级。第 11 条坑就是这么来的。

---

## 8. 与「盯盘小灯」的关系

两者是**互相独立的菜单栏小程序**，各占一个图标，互不干扰：

| | 盯盘小灯 | WB小灯 |
|---|---------|--------|
| 作用 | 看行情（股票/基金/可转债…） | 看 WorkBuddy 忙不忙 |
| 位置 | `Documents/Work/大AI开发/盯盘小灯/` | `Documents/Work/大AI开发/WB小灯/` |
| bundle id | `com.gangge.dingpan(.cn)` | `io.wblight.menubar` |

---

## 9. 后续可选增强

| 想做的事 | 思路 |
|---------|------|
| 菜单栏直接显示"忙了多久" | `self.item.button.attributedTitle` 里拼上时长数字 |
| 记住"今天总共忙了多久" | 在 `refresh` 里把 busy 时长累加进 `NSUserDefaults` |
| 允许手动钉住某个会话 | 现在是自动取 mtime 最新的 transcript；可加个菜单项手动指定会话 id |
| 红灯时也显示"上次干完是几点" | 面板里已有"活动于 X 前"，可再加一行"上次交付" |

> ⚠️ 别再做的一件事：**叠加进程 CPU 当辅助信号**。V1.2 实测过，深度思考时 19 个 WorkBuddy 进程的 CPU 增量全部 < 0.01s/s（推理在云端），这条路线是死的，见第 7 节坑 #10。
