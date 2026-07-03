# OPPS：开放软件包支付标准

## Open Package Payment Standard

- 版本： 1.1.0
- 状态： 草案
- 更新日期： 2026-07-03

---

前言

为什么需要 OPPS

开源软件支撑了现代互联网的基础设施，但开源作者的资金可持续性和尊严问题至今未被解决。

这里有两个伤口，不是只有一个。

第一个伤口是钱。捐赠不稳定，商业服务太复杂，平台抽成高。大多数作者一分钱也拿不到。

第二个伤口是尊重。代码跑在几千个项目里，支撑着几百亿的生意。但没有人知道作者的名字，没有人在乎作者的存在。偶尔有人来提 issue：“你这破库怎么还不更新？”

收入是一种尊重，但尊重不只有收入。

现有的解决方案要么依赖捐赠（不稳定，靠施舍心态），要么依赖商业服务（增加复杂度，把开源变成卖货），要么依赖平台绑定（缺乏通用性，作者被平台锁死）。

OPPS 不试图解决所有问题。它只做一件事：

为软件包提供一个可选的、标准的方式，让作者可以定义：用户在获取这个包之前，需要完成什么动作。

这个动作，可以是付钱，可以是点个 Star，可以是关注账号，可以说一声谢谢，也可以什么都不做。

OPPS 的核心理念是：

软件包除了描述“怎么安装”，还应该能够描述“怎么被认可”。

因为在一个健康的社会里，劳动应该被看见，价值应该被回报。而“回报”的形式，不应该由协议来规定。

---

设计目标

1. 协议中立 —— 不规定回报的形式。钱是回报，关注是回报，感谢也是回报。OPPS 只提供能力，不规定内容。
2. 极简设计 —— 协议只增加一个可选字段 acknowledgment.uri。
3. 平台解耦 —— 包管理器不处理任何回报逻辑，全部由外部插件完成。
4. 向后兼容 —— 没有 acknowledgment.uri 的包，行为与今天完全一致。
5. 安全可验证 —— 签名机制防止篡改，回报证明支持重复安装。
6. 生态可扩展 —— 任何包管理器、仓库、回报机制都可以兼容。

---

核心定义：什么是“支付”

在 OPPS 里，“支付”是一个广义概念。它指用户为获取软件包向作者付出的任何形式的回报。

传统定义里，支付 = 金钱转移。

OPPS 的定义是：

支付 = 金钱 + 关注 + 感谢 + 认可 + 任何作者认为重要的回报形式

它可以是：

- 金钱：acknowledgment://lib/1.0?amount=0.01&currency=USD
- 关注：acknowledgment://lib/1.0?action=follow&platform=github
- Star：acknowledgment://lib/1.0?action=star&repo=author/lib
- 感谢：acknowledgment://lib/1.0?action=thank&to=@author
- 实名背书：acknowledgment://lib/1.0?action=endorse&message=I+use+this
- 邮箱注册：acknowledgment://lib/1.0?action=register&email=required
- 企业审批：acknowledgment://lib/1.0?action=approval&enterprise=acme
- 什么都不做：没有这个字段，就是免费

每一种，都是支付。每一种，都是用户在向作者付出。

---

架构总览

```
                    ┌─────────────────┐
                    │      OPPS       │
                    └─────────────────┘
                             │
       ┌─────────────────────┼─────────────────────┐
       │                     │                     │
       ▼                     ▼                     ▼
┌─────────────┐    ┌─────────────────┐    ┌──────────────┐
│  Package    │    │   Repository    │    │   Acknowledgment│
│  Metadata   │    │    Protocol     │    │   Plugin     │
└─────────────┘    └─────────────────┘    └──────────────┘
       │                     │                     │
       └─────────────────────┼─────────────────────┘
                             ▼
                    ┌─────────────────┐
                    │  Package        │
                    │  Manager        │
                    └─────────────────┘
```

关键设计：回报不是包管理器的一部分——回报只是插件。

---

第一章：Package

1.1 包结构

一个符合 OPPS 标准的软件包结构如下：

```
hello/
├── src/
│   └── ...
├── package.toml
├── LICENSE
└── SIGNATURE
```

1.2 必须存在的文件

只有 package.toml 是必须解析的。

- LICENSE 和 SIGNATURE 用于安全验证。
- 未签名的包：安装时显示警告，但仍可安装（兼容模式）。

---

第二章：package.toml

2.1 格式

采用 TOML 格式，确保人类可读且易于解析。

2.2 完整示例

示例一：付费下载

```toml
[package]
name = "json"
version = "1.2.0"
description = "Fast JSON parser"
license = "MIT"

[source]
repository = "https://repo.opps.org/json"
homepage = "https://json.example.com"

[dependencies]
logger = "^2.0"

[acknowledgment]
uri = "acknowledgment://json/1.2.0?amount=0.01&currency=USD"
```

示例二：需要 Star

```toml
[package]
name = "logger"
version = "2.1.0"
description = "Simple logging library"
license = "Apache-2.0"

[acknowledgment]
uri = "acknowledgment://logger/2.1.0?action=star&repo=author/logger&platform=github"
```

示例三：需要说谢谢

```toml
[package]
name = "utils"
version = "0.5.0"
description = "Various utility functions"
license = "MIT"

[acknowledgment]
uri = "acknowledgment://utils/0.5.0?action=thank&to=@author_name"
```

示例四：免费

```toml
[package]
name = "free-lib"
version = "3.0.0"
description = "Completely free library"
license = "MIT"

# 没有 [acknowledgment] 字段
# 这就是免费——不需要任何回报
```

2.3 字段说明

| 字段 | 必填 | 说明 |
|------|------|------|
| package.name | ✅ | 包名，全局唯一 |
| package.version | ✅ | 语义化版本号 |
| package.description | 可选 | 简短描述 |
| package.license | 推荐 | 许可证标识 |
| source.repository | 推荐 | 源码仓库地址 |
| dependencies | 可选 | 依赖列表及版本约束 |
| acknowledgment.uri | 可选 | 回报授权 URI |

核心原则：

- 没有 acknowledgment.uri → 免费安装，不需要任何回报。
- 有 acknowledgment.uri → 需要通过回报插件完成动作后才能安装。

协议本身不包含价格、币种、收款账户、平台等信息——全部封装在 URI 中。

---

第三章：Acknowledgment URI

3.1 格式

```
acknowledgment://<identifier>?<parameters>
```

3.2 设计原则

OPPS 核心不解析 acknowledgment:// 协议的内容。

它只做一件事：

```
Plugin.Perform(uri) → Result
```

3.3 示例

```
# 金钱支付
acknowledgment://json/1.2.0?amount=0.01&currency=USD

# 订阅
acknowledgment://game-engine/3.0.0?amount=30.00&currency=USD&recurring=monthly

# 关注作者
acknowledgment://lib/2.0.0?action=follow&platform=github&user=author

# Star 项目
acknowledgment://tool/1.0.0?action=star&repo=author/tool

# 说谢谢
acknowledgment://helper/0.1.0?action=thank&to=@author

# 实名背书
acknowledgment://sdk/5.0.0?action=endorse&message=I+use+this+in+production

# 企业审批
acknowledgment://enterprise-lib/3.0.0?action=approval&enterprise=acme&department=engineering
```

3.4 说明

- 所有回报相关的信息（金额、币种、平台、动作类型）完全由 URI 承载。
- Core 不需要理解这些参数的含义，只负责传递给插件。
- 作者定义“要什么”，插件实现“怎么做”，Core 只是“传话的人”。

---

第四章：Acknowledgment Plugin

4.1 统一接口

所有回报插件必须实现同一个接口：

```typescript
interface AcknowledgmentPlugin {
  // 判断是否能处理该 URI
  CanHandle(uri: string): boolean;

  // 执行回报动作
  Perform(uri: string): Promise<AcknowledgmentResult>;
}
```

4.2 插件示例

- 支付宝插件 → 处理 acknowledgment://...?amount=...
- 微信支付插件 → 处理 acknowledgment://...?wechat=...
- Stripe 插件 → 处理 acknowledgment://...?stripe=...
- GitHub Star 插件 → 处理 acknowledgment://...?action=star...
- Twitter Follow 插件 → 处理 acknowledgment://...?action=follow&platform=twitter...
- Thank You 插件 → 处理 acknowledgment://...?action=thank...（弹出一个感谢框，用户点确认）
- 企业采购插件 → 处理 acknowledgment://...?action=approval...

4.3 核心原则

- Core 永远不需要修改。
- 新增回报方式只需要新增插件。
- 插件由用户自行安装和信任，OPPS 不强制捆绑任何平台。
- 金钱、关注、感谢、背书——在协议眼里，它们平等地都是“插件行为”。

---

第五章：Acknowledgment Result

5.1 允许的返回结果

回报插件只能返回以下三种结果：

| 结果 | 含义 | 下一步 |
|------|------|--------|
| SUCCESS | 回报动作完成 | 继续安装 |
| FAILED | 回报动作失败 | 终止安装，显示错误 |
| CANCELLED | 用户取消 | 终止安装，不显示错误 |

5.2 成功时的额外数据

SUCCESS 必须包含回报证明（Proof）：

```typescript
interface AcknowledgmentResult {
  status: 'SUCCESS' | 'FAILED' | 'CANCELLED';
  proof?: AcknowledgmentProof;  // 仅 SUCCESS 时存在
  error?: string;               // 仅 FAILED 时存在
}
```

---

第六章：Signature

6.1 必须签名的内容

签名必须覆盖以下文件的完整内容：

- package.toml
- LICENSE
- 所有源码文件的哈希值列表

6.2 签名的作用

任何以下修改都会导致签名失效：

- 修改 acknowledgment.uri
- 修改 dependencies
- 修改 version
- 修改任何源码文件
- 替换 LICENSE

6.3 签名验证流程

1. 读取 SIGNATURE 文件
2. 使用作者公钥验证签名
3. 计算 package.toml、LICENSE 和源码文件的哈希值
4. 与签名中存储的哈希值比对
5. 全部一致 → 签名有效 → 继续安装
6. 不一致 → 签名无效 → 拒绝安装（除非用户强制覆盖）

6.4 Fork 场景

允许 Fork，但修改 package.toml 后必须重新签名，否则安装失败。

---

第七章：Repository

7.1 职责

仓库不负责回报处理。

它只负责：

- 上传（Upload）
- 下载（Download）
- 搜索（Search）
- 镜像同步（Mirror）

7.2 设计原则

仓库甚至不需要知道 acknowledgment.uri 是否存在。

它只是一个文件存储和索引服务。

7.3 与镜像的关系

- 镜像完全合规，只需原样复制文件。
- 回报校验发生在用户本地，不依赖仓库。

---

第八章：Dependency

8.1 独立决策

每个依赖包独立决定是否包含 acknowledgment.uri。

示例依赖树：

```
A (acknowledgment.uri = 需要付费 → 收费)
├── B (没有 acknowledgment.uri → 免费)
├── C (acknowledgment.uri = 需要 Star → 关注型回报)
└── D (没有 acknowledgment.uri → 免费)
```

8.2 安装流程

1. 解析 package.toml 获取所有依赖。
2. 对每个依赖：检查是否有 acknowledgment.uri。
3. 有 → 调用回报插件完成动作。
4. 无 → 直接下载安装。
5. 全部完成后，生成 package.lock。

8.3 说明

- 用户不需要预先知道哪些依赖需要回报，安装时自动处理。
- 回报流程对用户透明，但必须用户确认（插件负责 UI）。
- 一个项目可能有的依赖要钱，有的要 Star，有的免费——每个独立处理。

---

第九章：Price

9.1 协议不知道价格

OPPS 协议不包含任何价格字段。

价格（或回报动作）完全包含在 acknowledgment.uri 中：

```
acknowledgment://json/1.2.0?amount=0.01&currency=USD
                           └──────────────────────┘
                              回报信息在此
```

9.2 设计理由

- Core 不需要理解价格——它只是传递 URI。
- 不同的回报方式可以有完全不同的参数结构。
- 作者可以自由定价或自由定义回报要求，无需等待协议更新。
- 回报要求绑定在特定版本上，版本升级可以调整。

---

第十章：Lock

10.1 package.lock 的作用

安装完成后，生成 package.lock 文件，记录：

- 包名和版本
- 哈希值（确保文件未被篡改）
- 签名验证结果
- 回报证明（Acknowledgment Proof）

10.2 重新安装

当 package.lock 存在且以下条件全部满足时：

- 依赖树未发生变化
- package.lock 中的签名和证明有效
- 所有文件哈希匹配

则无需重新执行回报动作。

10.3 示例

```toml
[[packages]]
name = "json"
version = "1.2.0"
hash = "sha256:abc123..."
signature_valid = true

[packages.acknowledgment_proof]
action = "payment"
transaction_id = "txn_20260703_abc"
timestamp = "2026-07-03T10:00:00Z"
plugin = "stripe"

[[packages]]
name = "logger"
version = "2.1.0"
hash = "sha256:def456..."
signature_valid = true

[packages.acknowledgment_proof]
action = "star"
platform = "github"
repo = "author/logger"
timestamp = "2026-07-03T10:00:05Z"
plugin = "github-star"
```

---

第十一章：Acknowledgment Proof

11.1 定义

回报动作成功后，插件返回回报证明，包含：

- 动作类型（payment、star、follow、thank 等）
- 平台/插件标识
- 交易号或动作 ID（由平台签发）
- 数字签名（由平台签发）
- 执行时间

11.2 验证流程

1. Core 保存回报证明到 package.lock。
2. 后续安装时，Core 验证：
   - 签名是否有效
   - 动作 ID 是否未被撤销（可选，依赖插件）
3. 验证通过 → 无需重新执行回报动作。

11.3 设计原则

- Core 不关心 Proof 的具体格式。
- 插件负责生成和验证 Proof。
- Core 只负责存储和传递。

---

第十二章：CLI

12.1 命令列表

| 命令 | 说明 |
|------|------|
| oppm install | 安装依赖（自动处理回报） |
| oppm remove | 移除包 |
| oppm publish | 发布包到仓库 |
| oppm verify | 验证签名和完整性 |
| oppm update | 更新依赖 |
| oppm search | 搜索包 |
| oppm cache | 管理缓存 |
| oppm doctor | 诊断环境问题 |

12.2 重要说明

没有回报相关的命令。

回报完全自动触发：

- oppm install 遇到 acknowledgment.uri 时自动调用插件。
- 用户不需要手动执行任何回报命令。

---

第十三章：Core

13.1 Core 的职责

OPPS Core 只负责：

1. 读取 package.toml
2. 解析依赖树
3. 验证签名
4. 检查是否存在 acknowledgment.uri
   - 没有 → 继续安装
   - 有 → 调用回报插件
5. 回报成功 → 继续安装
6. 回报失败 → 终止安装
7. 生成 package.lock

13.2 Core 不知道的事

Core 不知道：

- 支付宝
- 微信支付
- Stripe
- PayPal
- GitHub Star
- Twitter Follow
- 企业审批流
- 任何具体的回报形式

这种解耦保证了 OPPS 的通用性和长期稳定性。

Core 只知道一件事：有一个 URI，调插件，等结果。其他的一概不知，也一概不管。

---

第十四章：Fork

14.1 允许 Fork

OPPS 完全允许 Fork。

14.2 修改需要重新签名

如果修改了 package.toml，必须重新签名，否则安装失败。

14.3 完全 Fork 的包

如果你 Fork 了一个有回报要求的包，修改了 acknowledgment.uri 指向自己的账户，并重新签名发布：

- 新的包是你的版本
- 原作者的回报不受影响
- 用户可以选择信任你的版本

---

第十五章：免费

15.1 唯一的免费方式

免费只有一种方式：没有 acknowledgment.uri

15.2 不支持的“免费”变体

OPPS 协议不包含以下概念：

- Donation（捐赠）
- Enterprise（企业版）
- Subscription（订阅）
- Trial（试用）

这些全部由 acknowledgment.uri 和回报插件自行处理。

设计理由：保持协议极简。

---

第十六章：企业

16.1 企业无需特殊协议

企业场景不需要特殊协议支持。

16.2 企业内部采购

企业可以使用内部回报插件：

- 调用企业采购 SDK
- 通过内部审批流自动批准
- 不涉及个人银行卡或钱包

16.3 Core 不知情

Core 依然不知道“企业”这个概念。

它只是调用了一个插件，而这个插件恰好是企业内部开发的。

---

第十七章：安全模型

17.1 威胁模型

| 威胁 | 缓解方式 |
|------|----------|
| 篡改 acknowledgment.uri | 签名机制 + 哈希校验 |
| 篡改源码 | 签名机制 + 哈希校验 |
| 伪造回报证明 | 平台数字签名 |
| 重放回报证明 | 证明中包含时间戳 + 动作 ID 唯一性 |
| 绕过回报插件 | 插件由用户信任，Core 不绕行 |
| 恶意插件窃取信息 | 用户自行选择信任的插件 |

17.2 信任模型

- 用户信任回报插件（如官方 Stripe 插件、GitHub Star 验证插件）
- 用户信任作者公钥（通过 Web of Trust 或证书体系）
- OPPS Core 本身不需要信任任何第三方

---

第十八章：与现有包管理器的兼容性

18.1 兼容策略

OPPS 设计为可通过插件或扩展逐步兼容现有包管理器：

| 包管理器 | 兼容方式 |
|----------|----------|
| npm | 通过 npm install 钩子或自定义 registry |
| pip | 通过 pip install 的 --find-links 或自定义索引 |
| Cargo | 通过自定义 registry |
| Maven | 通过自定义 repository |
| Homebrew | 通过 formula 扩展 |

18.2 渐进式迁移

- 现有包不需要修改即可被 OPPS 兼容客户端读取。
- 没有 acknowledgment.uri 的包行为完全不变。
- 作者可以自由选择是否添加 acknowledgment.uri。

---

第十九章：RFC 规范拆分

建议将 OPPS 拆分为一系列独立的 RFC 文档：

| RFC 编号 | 标题 | 说明 |
|----------|------|------|
| RFC-0001 | Package Metadata | package.toml 格式、必填/可选字段定义 |
| RFC-0002 | Repository Protocol | 上传、下载、搜索、镜像 API 规范 |
| RFC-0003 | Signature | 数字签名算法、哈希规则、验证流程 |
| RFC-0004 | Acknowledgment Plugin API | 插件接口定义、返回值、错误码 |
| RFC-0005 | Acknowledgment URI | acknowledgment.uri 字段定义和通用要求 |
| RFC-0006 | Lock File | package.lock 格式、回报证明记录 |
| RFC-0007 | Security Model | 威胁模型、信任模型、最佳实践 |
| RFC-0008 | CLI Interface | 命令行工具规范 |

---

常见问题（FAQ）

Q1：这违反开源定义吗？

答：不违反。OPPS 是一个包管理协议，不是许可证。许可证仍然可以是 MIT、GPL 或 Apache。

OPPS 不限制任何人修改或再分发——它只是提供了一个在安装时向作者表达回报的方式。

Q2：这和捐赠有什么不同？

答：本质区别。

- 捐赠是事后的、自愿的、不稳定的。用户用完了，觉得好，可能捐，也可能不捐。
- OPPS 是事前的、自动化的、体系化的。用户在下载之前就完成回报动作——不管是付钱还是 Star。

捐赠靠施舍心态，OPPS 靠价值交换。

Q3：没人会付费/给 Star，怎么办？

答：这是市场选择问题，不是协议问题。

OPPS 只是提供一个“能力”，不是“强制”。如果作者认为自己的软件值得某种回报，可以选择要求回报；如果用户认为不值，可以选择不用。

竞争激烈的领域（如 JSON 解析）回报要求会趋向于零或极低。高度专业化的领域（如游戏引擎）更容易获得回报。

OPPS 本身不干预市场定价。

Q4：企业会 Fork 绕过吗？

答：Fork 是完全合规的，但必须重新签名。

企业可以 Fork 包，移除 acknowledgment.uri，重新签名，在企业内部使用。这完全符合开源精神。

如果企业选择这样做，说明要么这个包的回报要求不合理，要么企业愿意承担维护 Fork 版本的长期成本。

Q5：如果作者的回报插件停止了怎么办？

答：如果作者指定的回报方式不再可用（比如一个支付平台停止服务），用户仍然可以使用已有的 package.lock 和回报证明进行重新安装。

如果用户需要新版本，只能联系作者或寻找替代方案。

Q6：这不就是把“道德绑架”自动化了吗？

答：不是。道德绑架是“你应该给我钱/Star，虽然你没义务”。

OPPS 是“如果你想下载这个包，你需要先完成这个动作”。这是明确的交换条件，不是模糊的道德期待。

用户有完全的选择权：接受条件并下载，或者拒绝条件并不下载。不存在“绑架”。

Q7：这和 GitCoin / Polar / OpenCollective 有何不同？

答：它们是捐赠/资助平台（事后付费）。

OPPS 是安装时回报协议（事前回报）。

两者可以共存：

- 捐赠平台用于“感谢作者”。
- OPPS 用于“为获取软件而回报作者”。

---

总结

OPPS 的核心价值不在于“发明新的支付系统”，也不在于“推翻现有包管理器”。

它的价值在于两件事：

第一，将“回报”从“包管理器”中解耦，让回报成为一个独立于平台的插件化能力。

通过一个可选的 acknowledgment.uri 字段和统一的插件接口，OPPS 让每一位开发者都拥有获得回报的能力——无论是金钱、关注、还是感谢。

第二，重新定义了“支付”。

在 OPPS 里，支付不只是钱。它是用户在获取软件时向作者付出的任何形式的回报。

金钱是回报。Star 是回报。关注是回报。感谢也是回报。

这让开源作者第一次有了一个标准化的方式，让他们的劳动被看见、被认可、被回报。具体形式由作者决定，由插件执行，由 Core 调度。

无论你是一个渴望用代码养活自己的独立作者，还是一个只想被更多同行认识的开发者，还是一个什么都不想要、纯粹热爱分享的黑客——OPPS 都为你提供了同样的尊重：

选择的权利。

OPPS 不是终点。它只是一个开始。

---


- 版本： 1.1.0
- 状态： 草案
- 作者： live.aileen or 白芊昔
- 更新日期： 2026-07-03
- 链接： https://github.com/OPPS/OPPS
- 邮箱： live.aileen@outlook.com
- QQ:  3142116244
- QQ 群： 1046801466
---

