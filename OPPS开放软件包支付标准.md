# OPPS：开放软件包支付标准 

## Open Package Payment Standard

- 版本： 1.0.0
- 状态： 草案
- 更新日期： 2026-07-03

前言

为什么需要 OPPS

开源软件支撑了现代互联网的基础设施，但开源作者的资金可持续性问题至今未被解决。现有的解决方案要么依赖捐赠（不稳定），要么依赖商业服务（增加复杂度），要么依赖平台绑定（缺乏通用性）。

OPPS 不试图解决所有问题。它只做一件事：

为软件包提供一个可选的、标准的支付授权方式。

它的核心理念是：

软件包除了描述"怎么安装"，还应该能够描述"怎么授权和付款"。

但支付本身不与任何特定平台绑定，而是通过统一的插件接口实现。

---

设计目标

1. 协议中立 —— 不鼓励收费，也不鼓励免费，只提供能力。
2. 极简设计 —— 协议只增加一个可选字段 payment.uri。
3. 平台解耦 —— 包管理器不处理支付，支付由外部插件完成。
4. 向后兼容 —— 没有 payment.uri 的包，行为与今天完全一致。
5. 安全可验证 —— 签名机制防止篡改，支付证明支持重复安装。
6. 生态可扩展 —— 任何包管理器、仓库、支付平台都可以兼容。

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
│  Package    │    │   Repository    │    │   Payment    │
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

关键设计：支付不是包管理器的一部分——支付只是插件。

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

- LICENSE 和 SIGNATURE 用于安全验证（详见第六章）。
- 未签名的包：安装时显示警告，但仍可安装（兼容模式）。

---

第二章：package.toml

2.1 格式

采用 TOML 格式，确保人类可读且易于解析。

2.2 完整示例

```toml
[package]
name = "json"
version = "1.2.0"
description = "Fast JSON parser"
license = "OPPS-1.0"

[source]
repository = "https://repo.opps.org/json"
homepage = "https://json.example.com"

[dependencies]
logger = "^2.0"
network = ">=1.5"

[payment]
uri = "payment://json/1.2.0?amount=0.01&currency=USD"
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
| payment.uri | 可选 | 支付授权 URI |

核心原则：

- 没有 payment.uri → 免费安装。
- 有 payment.uri → 需要通过支付插件完成授权后才能安装。

协议本身 不包含 价格、币种、收款账户、支付平台等信息——全部封装在 URI 中。

---

第三章：Payment URI

3.1 格式

```
payment://<identifier>?<parameters>
```

3.2 设计原则

OPPS 核心 不解析 payment:// 协议的内容。

它只做一件事：

```
Plugin.Pay(uri) → Result
```

3.3 示例

```
payment://json/1.2.0?amount=0.01&currency=USD
payment://game-engine/3.0.0?amount=30.00&currency=USD&subscription=monthly
payment://enterprise-sdk/5.0.0?amount=100.00&currency=USD&license=per-seat
```

3.4 说明

- 所有支付相关的信息（金额、币种、收款方、平台）完全由 URI 承载。
- Core（包管理器核心）不需要理解这些参数的含义，只负责传递给插件。

---

第四章：Payment Plugin

4.1 统一接口

所有支付插件必须实现同一个接口：

```typescript
interface PaymentPlugin {
  // 判断是否能处理该 URI
  CanHandle(uri: string): boolean;

  // 执行支付
  Pay(uri: string): Promise<PaymentResult>;
}
```

4.2 插件示例

- 支付宝插件 → 处理 payment://...?alipay=...
- 微信支付插件 → 处理 payment://...?wechat=...
- Stripe 插件 → 处理 payment://...?stripe=...
- PayPal 插件 → 处理 payment://...?paypal=...
- 企业采购插件 → 处理 payment://...?enterprise=...

4.3 核心原则

- Core 永远不需要修改。
- 新增支付方式只需要新增插件。
- 插件由用户自行安装和信任，OPPS 不强制捆绑任何支付平台。

---

第五章：Payment Result

5.1 允许的返回结果

支付插件只能返回以下三种结果：

| 结果 | 含义 | 下一步 |
|------|------|--------|
| SUCCESS | 支付成功 | 继续安装 |
| FAILED | 支付失败 | 终止安装，显示错误 |
| CANCELLED | 用户取消 | 终止安装，不显示错误 |

5.2 成功时的额外数据

SUCCESS 必须包含 Payment Proof（详见第十一章）。

```typescript
interface PaymentResult {
  status: 'SUCCESS' | 'FAILED' | 'CANCELLED';
  proof?: PaymentProof;  // 仅 SUCCESS 时存在
  error?: string;        // 仅 FAILED 时存在
}
```

---

第六章：Signature

6.1 必须签名的内容

签名必须覆盖以下文件的 完整内容：

- package.toml
- LICENSE
- 所有源码文件的 哈希值列表

6.2 签名的作用

任何以下修改都会导致签名失效：

- 修改 payment.uri
- 修改 dependencies
- 修改 version
- 修改任何源码文件
- 替换 LICENSE

6.3 签名验证流程

1. 读取 SIGNATURE 文件。
2. 使用作者公钥验证签名。
3. 计算 package.toml、LICENSE 和源码文件的哈希值。
4. 与签名中存储的哈希值比对。
5. 全部一致 → 签名有效 → 继续安装。
6. 不一致 → 签名无效 → 拒绝安装（除非用户强制覆盖）。

6.4 Fork 场景

允许 Fork，但修改 package.toml 后 必须重新签名，否则安装失败。

---

第七章：Repository

7.1 职责

Repository（仓库）不负责支付。

它只负责：

- 上传（Upload）
- 下载（Download）
- 搜索（Search）
- 镜像同步（Mirror）

7.2 设计原则

仓库甚至不需要知道 payment.uri 是否存在。

它只是一个文件存储和索引服务。

7.3 与镜像的关系

- 镜像完全合规，只需原样复制文件。
- 支付校验发生在 用户本地，不依赖仓库。

---

第八章：Dependency

8.1 独立决策

每个依赖包 独立决定 是否包含 payment.uri。

示例依赖树：

```
A (payment.uri = payment://A/...  → 收费)
├── B (没有 payment.uri → 免费)
├── C (payment.uri = payment://C/...  → 收费)
└── D (没有 payment.uri → 免费)
```

8.2 安装流程

1. 解析 package.toml 获取所有依赖。
2. 对每个依赖：检查是否有 payment.uri。
3. 有 → 调用支付插件完成授权。
4. 无 → 直接下载安装。
5. 全部完成后，生成 package.lock。

8.3 说明

- 用户不需要预先知道哪些依赖收费，安装时自动处理。
- 支付流程对用户透明，但必须用户确认（插件负责 UI）。

---

第九章：Price

9.1 协议不知道价格

OPPS 协议 不包含任何价格字段。

价格完全包含在 payment.uri 中：

```
payment://json/1.2.0?amount=0.01&currency=USD
                  └────────────────────────┘
                       价格信息在此
```

9.2 设计理由

- Core 不需要理解价格——它只是传递 URI。
- 不同的支付平台可以有不同的定价逻辑。
- 作者可以自由定价，无需等待协议更新。
- 价格绑定在特定版本上，版本升级可以调整价格。

---

第十章：Lock

10.1 package.lock 的作用

安装完成后，生成 package.lock 文件，记录：

- 包名和版本
- 哈希值（确保文件未被篡改）
- 签名验证结果
- Payment Proof（支付证明）

10.2 重新安装

当 package.lock 存在且以下条件全部满足时：

- 依赖树未发生变化
- package.lock 中的签名和证明有效
- 所有文件哈希匹配

则无需重新支付。

10.3 示例

```toml
[[packages]]
name = "json"
version = "1.2.0"
hash = "sha256:abc123..."
signature_valid = true

[packages.payment_proof]
transaction_id = "txn_20260703_abc"
timestamp = "2026-07-03T10:00:00Z"
plugin = "alipay"
```

---

第十一章：Payment Proof

11.1 定义

支付成功后，插件返回 Payment Proof，包含：

- 交易号（Transaction ID）
- 数字签名（由支付平台签发）
- 付款时间
- 支付插件标识

11.2 验证流程

1. Core 保存 Payment Proof 到 package.lock。
2. 后续安装时，Core 验证：
   - 签名是否有效。
   - 交易号是否未被撤销（可选，依赖插件）。
3. 验证通过 → 无需重新支付。

11.3 设计原则

- Core 不关心 Proof 的具体格式（JSON、JWT、自定义二进制）。
- 插件负责生成和验证 Proof。
- Core 只负责存储和传递。

---

第十二章：CLI

12.1 命令列表

| 命令 | 说明 |
|------|------|
| opp install | 安装依赖（自动处理支付） |
| opp remove | 移除包 |
| opp publish | 发布包到仓库 |
| opp verify | 验证签名和完整性 |
| opp update | 更新依赖 |
| opp search | 搜索包 |
| opp cache | 管理缓存 |
| opp doctor | 诊断环境问题 |

12.2 重要说明

没有支付相关的命令。

支付完全自动触发：

- opp install 遇到 payment.uri 时自动调用插件。
- 用户不需要手动执行任何支付命令。

---

第十三章：Core

13.1 Core 的职责

OPPS Core 只负责：

1. 读取 package.toml
2. 解析依赖树
3. 验证签名
4. 检查是否存在 payment.uri
   - 没有 → 继续安装
   - 有 → 调用支付插件
5. 支付成功 → 继续安装
6. 支付失败 → 终止安装
7. 生成 package.lock

13.2 Core 不知道的事

Core 不知道：

- 支付宝
- 微信支付
- Stripe
- PayPal
- 银行卡
- 任何具体支付平台

这种解耦保证了 OPPS 的通用性和长期稳定性。

---

第十四章：Fork

14.1 允许 Fork

OPPS 完全允许 Fork。

14.2 修改需要重新签名

如果修改了 package.toml，必须重新签名，否则安装失败。

14.3 完全 Fork 的包

如果你 Fork 了一个收费包，修改了 payment.uri 指向自己的账户，并重新签名发布：

- 新的包是你的版本。
- 原作者的收费不受影响。
- 用户可以选择信任你的版本。

---

第十五章：免费

15.1 唯一的免费方式

免费只有一种方式：

没有 payment.uri

15.2 不支持的"免费"变体

OPPS 协议 不包含 以下概念：

- Donation（捐赠）
- Enterprise（企业版）
- Subscription（订阅）
- Trial（试用）

这些全部由 payment.uri 和支付插件自行处理。

设计理由：保持协议极简。

---

第十六章：企业

16.1 企业无需特殊协议

企业场景不需要特殊协议支持。

16.2 企业内部采购

企业可以使用内部支付插件：

- 调用企业采购 SDK。
- 通过内部审批流自动批准。
- 不涉及个人银行卡或钱包。

16.3 Core 不知情

Core 依然不知道 "企业" 这个概念。

它只是调用了一个插件，而这个插件恰好是企业内部开发的。

---

第十七章：安全模型

17.1 威胁模型

OPPS 考虑以下威胁：

| 威胁 | 缓解方式 |
|------|----------|
| 篡改 payment.uri | 签名机制 + 哈希校验 |
| 篡改源码 | 签名机制 + 哈希校验 |
| 伪造支付证明 | 支付平台数字签名 |
| 重放支付证明 | Proof 中包含时间戳 + 交易号唯一性 |
| 绕过支付插件 | 插件由用户信任，Core 不绕行 |
| 恶意插件窃取信息 | 用户自行选择信任的插件 |

17.2 信任模型

- 用户信任 支付插件（如官方支付宝插件）。
- 用户信任 作者公钥（通过 Web of Trust 或证书体系）。
- OPPS Core 本身不需要信任任何第三方。

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
- 没有 payment.uri 的包行为完全不变。
- 作者可以自由选择是否添加 payment.uri。

---

第十九章：RFC 规范拆分

建议将 OPPS 拆分为一系列独立的 RFC 文档，便于维护和讨论：

| RFC | 标题 | 说明 |
|-----|------|------|
| RFC-0001 | Package Metadata | package.toml 格式、必填/可选字段定义 |
| RFC-0002 | Repository Protocol | 上传、下载、搜索、镜像 API 规范 |
| RFC-0003 | Signature | 数字签名算法、哈希规则、验证流程 |
| RFC-0004 | Payment Plugin API | 插件接口定义、返回值、错误码 |
| RFC-0005 | Payment URI | payment.uri 字段定义和通用要求 |
| RFC-0006 | Lock File | package.lock 格式、支付证明记录 |
| RFC-0007 | Security Model | 威胁模型、信任模型、最佳实践 |
| RFC-0008 | CLI Interface | 命令行工具规范 |

---

常见问题（FAQ）

Q1：这违反开源定义吗？

答： 不违反。OPPS 是一个包管理协议，不是许可证。许可证仍然可以是 MIT、GPL 或 Apache。

OPPS 不限制任何人 "修改" 或 "再分发"——它只是提供了一个在安装时进行支付授权的方式。

Q2：没人会付费，怎么办？

答： 这是市场选择问题，不是协议问题。

OPPS 只是提供一个"能力"（Capability），不是"强制"（Mandate）。如果作者认为自己的软件有价值，可以选择收费；如果用户认为不值，可以选择不用。

Q3：企业会 Fork 绕过吗？

答： Fork 是完全合规的，但必须重新签名。

企业可以 Fork 包，移除 payment.uri，重新签名，在企业内部使用。这完全符合开源精神。

如果企业选择这样做，说明：要么这个包的价值不足以让企业付费，要么企业愿意承担维护 Fork 版本的长期成本。

Q4：有人会破解吗？

答： 任何软件都可能被破解。

OPPS 的目标不是 "完美防破解"，而是：

1. 让合规支付变得极其方便。
2. 让绕过支付变得明显（签名失效、需要重新签名）。
3. 通过社会共识和法律手段保护作者权益。

Q5：这会导致依赖越来越贵吗？

答： 市场会自然调节。

竞争激烈的领域（如 JSON 解析、日志库）价格会趋向于零或极低。
高度专业化的领域（如游戏引擎、AI 框架）更容易获得付费。

OPPS 本身不干预市场定价。

Q6：如果作者的支付插件停止了怎么办？

答： 如果作者停止维护支付插件，用户仍然可以使用已有的 package.lock 和 Payment Proof 进行重新安装。

如果用户需要新版本，只能联系作者或寻找替代方案。

Q7：这与 GitCoin / Polar / OpenCollective 有何不同？

答： 它们是 捐赠/资助平台（事后付费）。

OPPS 是 安装时支付授权协议（事前付费）。

两者可以共存：

- 捐赠平台用于 "感谢作者"。
- OPPS 用于 "为获取软件授权而支付"。

---

总结

OPPS 的核心价值不在于 "发明新的支付系统"，也不在于 "推翻现有包管理器"。

它的价值在于：

将"支付"从"包管理器"中解耦，让支付成为一个独立于平台的插件化能力。

通过一个可选的 payment.uri 字段和统一的插件接口，OPPS 让每一位开发者都拥有获得收入的能力，同时保留了完全免费的选择。

无论你是一个独立的开源作者，还是大型企业的架构师，OPPS 都为你提供了一个 中立、开放、可扩展 的软件包支付标准。

---

OPPS 不是终点。它只是一个开始。

---

- 版本： 1.0.0
- 状态： 草案
- 作者： live.aileen or 白芊音
- 更新日期： 2026-07-03
- 链接： https://github.com/OPPS/OPPS
- 邮箱： live.aileen@outlook.com
- QQ:  3142116244
- QQ 群： 1046801466
---