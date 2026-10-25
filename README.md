# 蜂群僚机 · HiveWing

> 一个跑在你自己电脑上的「聊天参谋」：把她的消息丢进来，蜂群给你三条**能直接发出去**的话。
> 接 QQ / 微信 / 抖音，电脑和手机实时互通，模型可以用 DeepSeek，也可以完全用本地 Ollama。

```
她：这周终于要忙完了
────────────────────────────────────────
局势   分享日常 · 她在主动给你话题入口
目标   顺着细节聊深一点
打法   接细节、轻轻推

① 84.8  终于解放了，那得庆祝一下          [发送] [复制]
② 84.5  那周末出来透透气？我带你去个地方    [发送] [复制]
③ 82.9  这周辛苦，收尾了吗                [发送] [复制]
```

不是话术库。它的核心是**先判断局势，再决定这一步要干什么，最后用你自己的语气写出来**——所以有「去 AI 味」引擎、有反 PUA 质检、有你的历史消息做风格样本。

---

## 它到底做了什么

| 能力 | 说明 |
|---|---|
| 蜂群调度 | 感知 → 侦察 → 定策 → 起稿（并行）→ 打磨 → 质检，按局势自动选「小分队 / 标准 / 深潜」三档队形 |
| 层级处理 | L0 女王分诊 → L1 感知/侦察 → L2 军师/质检 → L3 起稿/打磨（多工蜂并行） |
| 口语化 | 确定性改写 + AI 痕迹评分（连接词、书面词、长句、排比、客服腔…），改差不要 |
| 风格迁移 | 拿你自己发过的话当 few-shot，写出来像你，不像模型 |
| 反模式闸门 | 舔狗、鸡汤、说教、炫富、卖惨、道德绑架、越界、考验人性 → 直接标红，默认禁止发送 |
| 记忆 | 每个联系人的关系阶段、已知事实（工作/宠物/生日/忌口）、历史打法胜率 |
| 三端接入 | QQ（OneBot）、微信（wxauto）、抖音（剪贴板桥 + 注入），外加万能「手动/复制」 |
| 双端互通 | 电脑起稿，手机批准；手机收消息，电脑发送。同一个 SQLite，同一条 WebSocket |
| 双模型 | DeepSeek 云 + Ollama 本地，按角色分档；云端挂了自动降级到本地/离线引擎 |

---

## 30 秒跑起来

Windows，Python 3.11+（本项目在 3.13 上开发）。

```powershell
cd hivewing
python -m pip install -r requirements.txt
python -m hivewing serve
```

会自动打开 `http://127.0.0.1:8760/?token=...`，并打印手机访问地址。
**没配任何模型也能用**：默认走离线引擎，能完整体验流程和界面。

自检：

```powershell
python -m hivewing doctor
```

---

## 配模型

### DeepSeek（推荐，效果最好）

```powershell
$env:DEEPSEEK_API_KEY = "sk-..."
python -m hivewing serve
```

或者在界面 ⚙ →「模型」里填 Key。key 只存在本机 `config.yaml`，接口返回时会被打码。

成本参考：一轮标准起稿约 7 次调用、1 万 token 上下，按 DeepSeek 价格约几分钱。

### Ollama（本地免费）

```powershell
ollama pull qwen2.5:7b
ollama serve
```

界面 ⚙ →「模型」把主模型切成 Ollama，快档建议 `qwen2.5:3b`（感知/侦察用），强档 `qwen2.5:7b`。
小模型写中文容易端着，所以「打磨蜂」这一步会做确定性的口语化兜底。

### 角色分档（config.yaml）

```yaml
llm:
  routing:
    perceive: fast     # 判断局势，用小模型
    scout: fast        # 抽事实，用小模型
    strategize: smart  # 定打法，用强模型
    draft: smart       # 写原话，用强模型
    polish: fast       # 去 AI 味，小模型够
    critic: smart      # 打分挑刺，要准
```

---

## 接入三个平台

三种接法，从稳到「黑科技」：

### 1. 手动 / 剪贴板（零配置，所有平台都能用）

界面里点「她刚发来」→ 粘她的消息 → 蜂群起稿 → 点「复制」→ 自己粘进聊天框。

### 2. 全局热键：Ctrl+Alt+H（推荐给抖音）

在**任何**聊天窗口里复制她那条消息，按 `Ctrl+Alt+H`：
自动归到你当前选中的会话 → 蜂群起稿 → 手机/电脑同时弹出草稿。

发送走「通用注入」：点发送后，程序找到微信/QQ/抖音的窗口，把文字粘进输入框（不自动回车，你自己扫一眼再发）。

### 3. 协议接入

**QQ（OneBot 11 / NapCat）**

1. 装 [NapCat](https://github.com/NapNeko/NapCatQQ)，登录一个小号
2. NapCat 配置里开 HTTP 服务（默认 `http://127.0.0.1:3000`）和反向 WebSocket
3. 反向 WS 地址填 `ws://127.0.0.1:8760/onebot/ws`
4. 界面 ⚙ →「接入」→ 启动 QQ，填 HTTP 地址和 token

只处理**私聊**消息，群消息不进（这是撩妹工具，不是群管机器人）。

**微信（wxauto）**

```powershell
python -m pip install wxauto
```

保持 PC 微信登录、窗口不要最小化，⚙ →「接入」→ 启动微信。
wxauto 走 Windows UI 自动化，不同版本 API 有差异，代码做了兼容；不工作时看适配器状态里的报错。

**抖音**

没有开放的私聊 API，也不建议用逆向协议（封号 + 不稳定）。
用「复制 + Ctrl+Alt+H + 通用注入」，体验和原生接入几乎一样。

> 非官方通道都有账号风险，**先拿小号试**。

---

## 手机上用

1. 电脑和手机连同一个 Wi-Fi
2. 界面 ⚙ →「手机配对」→ 扫码
3. 加到主屏幕，就是一个全屏 App

手机能干的事：看会话、看她刚发的消息、看蜂群轨迹、批准/修改/复制草稿、自己写一句发出去。
手机上的操作会实时同步到电脑（反之亦然）——两边都只是这个 Hub 的客户端。

> 如果打不开：Windows 防火墙放行 8760 端口。服务默认监听 `0.0.0.0`（手机要连），但所有业务接口都要 token。

---

## 蜂群是怎么分工的

```
                    ┌──────── L0 女王 ────────┐
                    │ 分诊：这条消息值多少蜂？ │
                    │ 小分队 / 标准 / 深潜     │
                    └───────────┬────────────┘
              ┌─────────────────┴─────────────────┐
        L1 感知蜂                             L1 侦察蜂
     意图 / 情绪 / 潜台词                  事实 / 话头 / 雷区
              └─────────────────┬─────────────────┘
                          L2 军师蜂
                    目标 / 2-3 个打法 / 禁忌
                                │
              ┌─────────────────┼─────────────────┐
         L3 起稿蜂①         起稿蜂②          起稿蜂③
         （接梗）           （关心）          （给台阶）
              └─────────────────┼─────────────────┘
                          L3 打磨蜂
                   确定性改写 + 去 AI 味 + 复盘
                                │
                          L2 质检蜂
              五项打分 + 反模式扫描 + 返工批注
                                │
                       排名 / 返工 / 呈报
```

四个让它真的像「蜂群」而不是「一条 prompt 链」的机制：

1. **信息素**：每种打法（给台阶邀约、先接情绪…）按意图记录「用了几次、成了几次」，下次优先高胜率的。
2. **并行**：感知和侦察同时出发；3-4 只起稿蜂同时写不同角度，温度也不同。
3. **闭环返工**：质检给了 revise，女王带着批注再派一轮，最多 1-2 次。
4. **预算**：默认一轮最多 18 次调用 / 75 秒。超了不是报错，而是拿现有最好的稿子交。

轨迹在界面右下角实时滚动，每一步的模型、耗时、结论都能看见。

---

## 技能包

内置两个：

- **tong-jincheng**：来自 [hotcoffeeshake/tong-jincheng-skill](https://github.com/hotcoffeeshake/tong-jincheng-skill)（MIT）。用它的五个心智模型（吸引力 > 讨好、给台阶、人性不可考验、炫耀即暴露）和九条决策启发式做**判断**，用它的表达 DNA 做**语气约束**。
- **hivewing-core**：自研方法论。总纲、破冰、接话三步、邀约公式、冷场救场、已读不回、情绪价值、聊天节奏、边界感、口语化对照表。

**导入你自己的技能包**：任何 GitHub 上的 `SKILL.md` 仓库都行。

```powershell
python -m hivewing skill import https://github.com/owner/repo
```

或者在 ⚙ →「技能包」里填 URL。技能按关键词检索，每次只把最相关的几节塞进提示词，不会把整个仓库灌进去。

想改内容？直接编辑 `skills/packs/<名字>/SKILL.md`，`##` 标题就是检索单元。`BRIEF.md` 是每次都会带上的精简版。

---

## 去 AI 味引擎

`hivewing/polish.py`，两条路径：

**tidy()** 确定性改写（等义替换，不改意思）：

| 改前 | 改后 |
|---|---|
| 首先，我非常开心！但是你是否方便？ | 挺高兴的。不过你有空吗 |
| 因此／此外／以及／进行 | 所以／还有／和／做 |
| 希望你今天顺利 | 今天顺点 |

**lint()** 打分 0-100，检出的问题：书面连接词、AI 套话、助手腔、客服腔、长句、连环问号、感叹号堆叠、排比、空泛鼓励、过度客气、表情刷屏、句句「我」开头、模糊词堆叠。

打磨蜂改完会再跑一次 lint，**分不如原来就丢弃**——保证稿子只会变好，不会被模型改坏。

---

## 反模式闸门

`hivewing/critic.py` 里 12 类规则，灵感直接来自童锦程 skill 的「绝对反对」清单：

舔狗式讨好 · 连发轰炸 · 鸡汤 · 说教 · 炫耀 · 卖惨 · 道德绑架 · 油腻称呼 · 查户口 · 考验人性 · 越界 · 急着要结果

命中 `blocking` 的稿子：标记 ⛔、评分压到 34 以下、`/send` 直接拒绝。
其余命中只扣分并在卡片上提示，你自己判断。

这条闸门不是为了政治正确——**这些话本身就是最容易把关系聊死的**。

---

## 目录

```
情感辅助/
├─ hivewing/
│  ├─ swarm/          queen.py 调度核心，prompts.py 各蜂提示词
│  ├─ llm/            deepseek / ollama / mock / router（分档 + 降级）
│  ├─ adapters/       manual / inject / onebot / wxauto / douyin / bridge
│  ├─ skills/         loader.py（frontmatter + 检索 + GitHub 导入）
│  ├─ server/         app.py REST+WS，hub.py 消息总线，static/ 前端
│  ├─ polish.py       口语化与 AI 痕迹
│  ├─ critic.py       反模式
│  ├─ context.py      上下文与风格样本
│  ├─ db.py           SQLite
│  └─ __main__.py     CLI
├─ skills/packs/      tong-jincheng/ · hivewing-core/
├─ tests/             17 个测试，全离线
├─ docs/              ARCHITECTURE.md · ADAPTERS.md · ETHICS.md
└─ config.yaml        首次运行自动生成（含随机 token）
```

---

## 常用命令

```powershell
python -m hivewing serve                 # 启动（PC + 手机）
python -m hivewing serve --offline       # 强制离线引擎
python -m hivewing serve --port 9000     # 换端口
python -m hivewing doctor                # 自检环境
python -m hivewing compose --contact 小雅 --text "在干嘛"   # 命令行起稿
python -m hivewing skill list            # 看技能包
python -m hivewing seed                  # 灌演示数据
python -m unittest discover -s tests -v  # 跑测试
```

---

## 接口

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/api/bootstrap` | 首屏全量状态（含隐私状态） |
| POST | `/api/messages` | 收一条消息（会触发蜂群） |
| POST | `/api/contacts/{id}/compose` | 起稿（`sync:true` 同步返回） |
| POST | `/api/drafts/{id}/send` | 发送 / 复制，必须带 `confirm:true` |
| POST | `/api/privacy/consent` | 记录授权范围（必选 `read_chat`、`send_confirm`） |
| POST | `/api/privacy/revoke` | 撤回授权，同时停掉所有读取通道 |
| POST | `/api/privacy/purge` | 清数据（`messages` / `all`） |
| GET | `/api/wechat/screen` | 读微信窗口当前聊天（OCR，本机实验；要 `read_chat` 授权） |
| PUT | `/api/settings` | 改人设、语气、模型、接入；改不动授权字段 |
| POST | `/api/skills/import` | 从 GitHub 导入技能包 |
| WS | `/ws?token=` | 实时事件：`message.new` / `trace` / `draft.ready` / `privacy` |
| WS | `/onebot/ws` | NapCat 反向 WS |

除 `/api/health` 和静态资源外，全部要 `X-Hive-Token` 头或 `?token=`。token 在 `config.yaml`，首次启动自动生成。

---

## 隐私：默认一个字节都不出这台电脑

第一次启动接入通道时，会弹一次授权页，四项清清楚楚：

| 项 | 说明 | 能否拒绝 |
|---|---|---|
| 读取聊天内容 | 读微信/QQ/抖音窗口里她发来的消息 | 必选 |
| 本机保存 | 记录存在 `data/hivewing.db` | 可选 |
| 发给云端模型 | 把最近几条对话发给 DeepSeek | **可选，默认不勾** |
| 每条都要我确认 | 蜂群可以自动起稿，但不能自动发 | 必选，锁死 |

**没勾「发给云端模型」会怎样？** 模型路由直接降级到本地离线引擎，DeepSeek 一次都不会被调用。勾了之后还要把「只用本机模型」关掉，才会真的出机——两道开关，不靠记性。

**出机前脱敏**：手机号、身份证、银行卡、邮箱、门牌地址、车牌，在送进提示词之前就被替换成 `[手机号]` 这类占位符。打码发生在内存里，不写日志。

**每条发送都要你点**：接口层面强制 `confirm: true`，没确认直接 428。界面上的确认框会把「发给谁 / 哪个平台 / 走哪条通道 / 发什么」摆在你面前，点下去才真的发。这个开关在设置接口里改不动。

**审计**：授权、撤回、每次发送、每次模型调用（去哪、是否脱敏）都记进 `events` 表，「设置 → 隐私」里能看到最近 40 条。

**清空**：清空聊天记录 / 清空全部数据 / 撤回授权，三个按钮都在「设置 → 隐私」。也可以直接删 `data/` 目录。

> 技术上能做的都做了，但有一件事工具替不了你：**她不知道你在用工具**。要不要说、什么时候说，是你的事。我的建议是关系稳定之后自己讲，别等她从别的地方发现。

---

## 边界（认真读一遍）

1. **默认人工确认**。`require_human_approval` 和「发送前检查」是故意的：让 AI 全自动代聊，等于让对方跟一个不存在的人谈恋爱，短期有效、长期必翻车。你要的是把话说到位，不是把号交出去。
2. **它是参谋，不是替身**。风格样本取自你，观点得是你自己的。稿子读一遍，不像你就改。
3. **不碰未成年人、不碰骚扰、不碰已婚**。她说「不」就是「不」。
4. **非官方协议有封号风险**，小号先试。
5. 抖音没有官方私聊 API，别信任何「抖音协议机器人」，基本都是盗号或封号。

---

## 第三方

- [hotcoffeeshake/tong-jincheng-skill](https://github.com/hotcoffeeshake/tong-jincheng-skill) — MIT，已 vendor 到 `skills/packs/tong-jincheng/`，`BRIEF.md` 为二次蒸馏。
- [davidshimjs/qrcodejs](https://github.com/davidshimjs/qrcodejs) — MIT，手机配对二维码。
- 图标为内置 SVG，无外部依赖。

本项目代码 MIT。
