# OPPS: Open Package Payment Standard

## Open Package Payment Standard

- Version: 1.1.0
- Status: Draft
- Updated: 2026-07-03

---

Preface

Why OPPS is Needed

Open source software powers the infrastructure of the modern internet, yet the financial sustainability and dignity of open source authors remain unresolved.

There are two wounds, not just one.

The first wound is money. Donations are unstable, commercial services are too complex, and platforms take high cuts. Most authors receive nothing.

The second wound is respect. Code runs in thousands of projects, supporting billions of dollars in business. But no one knows the author's name. No one cares about the author's existence. Occasionally someone files an issue: "Why hasn't your crappy library been updated?"

Income is a form of respect, but respect is not only income.

Existing solutions either rely on donations (unstable, charity mentality), commercial services (added complexity, turning open source into a sales channel), or platform lock-in (lack of universality, authors trapped by platforms).

OPPS does not attempt to solve all problems. It does only one thing:

Provide packages with an optional, standard way for authors to define what action a user must complete before obtaining the package.

That action can be paying money, giving a Star, following an account, saying thank you, or doing nothing at all.

OPPS's core philosophy is:

A software package should not only describe "how to install", but also "how to be acknowledged".

Because in a healthy society, labor should be seen, and value should be rewarded. And the form of "reward" should not be dictated by a protocol.

---

Design Goals

1. Protocol-neutral — Does not prescribe the form of reward. Money is reward, attention is reward, gratitude is reward. OPPS only provides the capability, not the content.
2. Minimal design — The protocol adds only one optional field: acknowledgment.uri.
3. Platform-decoupled — Package managers do not handle any reward logic; everything is handled by external plugins.
4. Backward compatible — Packages without acknowledgment.uri behave exactly as they do today.
5. Secure and verifiable — Signature mechanism prevents tampering; reward proofs support reinstallation.
6. Ecologically extensible — Any package manager, repository, or reward mechanism can be compatible.

---

Core Definition: What is "Payment"

In OPPS, "payment" is a broad concept. It refers to any form of reward a user gives to an author in exchange for obtaining a software package.

In the traditional definition, payment = money transfer.

OPPS's definition is:

Payment = Money + Attention + Gratitude + Recognition + Any form of reward the author considers important

It can be:

- Money: acknowledgment://lib/1.0?amount=0.01&currency=USD
- Follow: acknowledgment://lib/1.0?action=follow&platform=github
- Star: acknowledgment://lib/1.0?action=star&repo=author/lib
- Thanks: acknowledgment://lib/1.0?action=thank&to=@author
- Endorsement: acknowledgment://lib/1.0?action=endorse&message=I+use+this
- Email registration: acknowledgment://lib/1.0?action=register&email=required
- Enterprise approval: acknowledgment://lib/1.0?action=approval&enterprise=acme
- Do nothing: the absence of this field means free

Each of these is a payment. Each is the user giving something to the author.

---

Architecture Overview

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

Key design: Reward is not part of the package manager — reward is just a plugin.

---

Chapter 1: Package

1.1 Package Structure

An OPPS-compliant software package has the following structure:

```
hello/
├── src/
│   └── ...
├── package.toml
├── LICENSE
└── SIGNATURE
```

1.2 Required Files

Only package.toml is mandatory for parsing.

- LICENSE and SIGNATURE are used for security verification.
- Unsigned packages: show a warning during installation, but can still be installed (compatibility mode).

---

Chapter 2: package.toml

2.1 Format

Uses TOML format for human readability and easy parsing.

2.2 Complete Examples

Example 1: Paid Download

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

Example 2: Star Required

```toml
[package]
name = "logger"
version = "2.1.0"
description = "Simple logging library"
license = "Apache-2.0"

[acknowledgment]
uri = "acknowledgment://logger/2.1.0?action=star&repo=author/logger&platform=github"
```

Example 3: Thank You Required

```toml
[package]
name = "utils"
version = "0.5.0"
description = "Various utility functions"
license = "MIT"

[acknowledgment]
uri = "acknowledgment://utils/0.5.0?action=thank&to=@author_name"
```

Example 4: Free

```toml
[package]
name = "free-lib"
version = "3.0.0"
description = "Completely free library"
license = "MIT"

# No [acknowledgment] field
# This is free — no reward required
```

2.3 Field Reference

| Field | Required | Description |
|-------|----------|-------------|
| package.name | ✅ | Package name, globally unique |
| package.version | ✅ | Semantic version number |
| package.description | Optional | Brief description |
| package.license | Recommended | License identifier |
| source.repository | Recommended | Source repository URL |
| dependencies | Optional | Dependency list with version constraints |
| acknowledgment.uri | Optional | Reward authorization URI |

Core principles:

- No acknowledgment.uri → Free installation, no reward required.
- Has acknowledgment.uri → Reward action must be completed via a plugin before installation.

The protocol itself does not contain price, currency, payment account, platform, etc. — all encapsulated in the URI.

---

Chapter 3: Acknowledgment URI

3.1 Format

```
acknowledgment://<identifier>?<parameters>
```

3.2 Design Principles

The OPPS core does NOT parse the acknowledgment:// protocol content.

It does only one thing:

```
Plugin.Perform(uri) → Result
```

3.3 Examples

```
# Money payment
acknowledgment://json/1.2.0?amount=0.01&currency=USD

# Subscription
acknowledgment://game-engine/3.0.0?amount=30.00&currency=USD&recurring=monthly

# Follow author
acknowledgment://lib/2.0.0?action=follow&platform=github&user=author

# Star project
acknowledgment://tool/1.0.0?action=star&repo=author/tool

# Say thanks
acknowledgment://helper/0.1.0?action=thank&to=@author

# Endorsement
acknowledgment://sdk/5.0.0?action=endorse&message=I+use+this+in+production

# Enterprise approval
acknowledgment://enterprise-lib/3.0.0?action=approval&enterprise=acme&department=engineering
```

3.4 Notes

- All reward-related information (amount, currency, platform, action type) is entirely carried by the URI.
- Core does not need to understand these parameters; it only passes them to the plugin.
- The author defines "what is wanted", the plugin implements "how to do it", and Core is just the "messenger".

---

Chapter 4: Acknowledgment Plugin

4.1 Unified Interface

All reward plugins must implement the same interface:

```typescript
interface AcknowledgmentPlugin {
  // Determines whether this plugin can handle the URI
  CanHandle(uri: string): boolean;

  // Executes the reward action
  Perform(uri: string): Promise<AcknowledgmentResult>;
}
```

4.2 Plugin Examples

- Alipay plugin → handles acknowledgment://...?amount=...
- WeChat Pay plugin → handles acknowledgment://...?wechat=...
- Stripe plugin → handles acknowledgment://...?stripe=...
- GitHub Star plugin → handles acknowledgment://...?action=star...
- Twitter Follow plugin → handles acknowledgment://...?action=follow&platform=twitter...
- Thank You plugin → handles acknowledgment://...?action=thank... (shows a thank-you dialog, user confirms)
- Enterprise procurement plugin → handles acknowledgment://...?action=approval...

4.3 Core Principles

- Core never needs to be modified.
- Adding a new reward method only requires adding a new plugin.
- Plugins are installed and trusted by the user; OPPS does not bundle any platform.
- Money, attention, gratitude, endorsement — in the protocol's eyes, they are all equally "plugin behaviors".

---

Chapter 5: Acknowledgment Result

5.1 Allowed Results

A reward plugin can only return the following three results:

| Result | Meaning | Next Step |
|--------|---------|-----------|
| SUCCESS | Reward action completed | Continue installation |
| FAILED | Reward action failed | Abort installation, show error |
| CANCELLED | User cancelled | Abort installation, no error |

5.2 Additional Data on Success

SUCCESS must include a reward proof:

```typescript
interface AcknowledgmentResult {
  status: 'SUCCESS' | 'FAILED' | 'CANCELLED';
  proof?: AcknowledgmentProof;  // Only present on SUCCESS
  error?: string;               // Only present on FAILED
}
```

---

Chapter 6: Signature

6.1 What Must Be Signed

The signature must cover the full content of the following files:

- package.toml
- LICENSE
- Hash list of all source files

6.2 Purpose of Signature

Any of the following modifications will invalidate the signature:

- Modifying acknowledgment.uri
- Modifying dependencies
- Modifying version
- Modifying any source file
- Replacing LICENSE

6.3 Signature Verification Process

1. Read the SIGNATURE file
2. Verify the signature using the author's public key
3. Compute hashes of package.toml, LICENSE, and source files
4. Compare with the hashes stored in the signature
5. All match → signature valid → continue installation
6. Mismatch → signature invalid → refuse installation (unless user forcibly overrides)

6.4 Fork Scenario

Forking is allowed, but modifying package.toml requires re-signing, otherwise installation fails.

---

Chapter 7: Repository

7.1 Responsibilities

The repository does NOT handle reward processing.

It is only responsible for:

- Upload
- Download
- Search
- Mirror synchronization

7.2 Design Principles

The repository does not even need to know whether acknowledgment.uri exists.

It is merely a file storage and indexing service.

7.3 Relationship with Mirrors

- Mirrors are fully compliant — they only need to copy files as-is.
- Reward verification happens on the user's local machine, independent of the repository.

---

Chapter 8: Dependency

8.1 Independent Decision

Each dependency independently decides whether to include acknowledgment.uri.

Example dependency tree:

```
A (acknowledgment.uri = requires payment → paid)
├── B (no acknowledgment.uri → free)
├── C (acknowledgment.uri = requires Star → attention-based reward)
└── D (no acknowledgment.uri → free)
```

8.2 Installation Process

1. Parse package.toml to get all dependencies.
2. For each dependency: check if acknowledgment.uri exists.
3. If yes → call the reward plugin to complete the action.
4. If no → download and install directly.
5. After all are complete, generate package.lock.

8.3 Notes

- The user does not need to know in advance which dependencies require reward; it is handled automatically during installation.
- The reward process is transparent to the user but requires user confirmation (the plugin handles the UI).
- A project may have some dependencies that require money, some that require a Star, some that are free — each is handled independently.

---

Chapter 9: Price

9.1 The Protocol Does Not Know About Price

The OPPS protocol does not contain any price fields.

Price (or reward action) is entirely contained within the acknowledgment.uri:

```
acknowledgment://json/1.2.0?amount=0.01&currency=USD
                           └──────────────────────┘
                               Reward info here
```

9.2 Design Rationale

- Core does not need to understand price — it only passes the URI.
- Different reward methods can have completely different parameter structures.
- Authors can freely set prices or define reward requirements without waiting for protocol updates.
- Reward requirements are bound to specific versions; version upgrades can adjust them.

---

Chapter 10: Lock

10.1 Purpose of package.lock

After installation, a package.lock file is generated, recording:

- Package name and version
- Hash value (ensures files have not been tampered with)
- Signature verification result
- Reward proof (Acknowledgment Proof)

10.2 Reinstallation

When package.lock exists and all of the following conditions are met:

- Dependency tree has not changed
- Signatures and proofs in package.lock are valid
- All file hashes match

Then no reward action needs to be re-executed.

10.3 Example

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

Chapter 11: Acknowledgment Proof

11.1 Definition

After a reward action succeeds, the plugin returns a reward proof, containing:

- Action type (payment, star, follow, thank, etc.)
- Platform/plugin identifier
- Transaction ID or action ID (issued by the platform)
- Digital signature (issued by the platform)
- Execution timestamp

11.2 Verification Process

1. Core saves the reward proof to package.lock.
2. On subsequent installations, Core verifies:
   - Whether the signature is valid
   - Whether the action ID has been revoked (optional, depends on the plugin)
3. Verification passes → no need to re-execute the reward action.

11.3 Design Principles

- Core does not care about the specific format of the Proof.
- The plugin is responsible for generating and verifying the Proof.
- Core is only responsible for storage and forwarding.

---

Chapter 12: CLI

12.1 Command List

| Command | Description |
|---------|-------------|
| oppm install | Install dependencies (auto-handles rewards) |
| oppm remove | Remove a package |
| oppm publish | Publish a package to the repository |
| oppm verify | Verify signature and integrity |
| oppm update | Update dependencies |
| oppm search | Search for packages |
| oppm cache | Manage cache |
| oppm doctor | Diagnose environment issues |

12.2 Important Notes

There are no reward-related commands.

Rewards are fully automatic:

- oppm install automatically calls the plugin when it encounters acknowledgment.uri.
- The user does not need to manually execute any reward commands.

---

Chapter 13: Core

13.1 Core Responsibilities

OPPS Core is only responsible for:

1. Reading package.toml
2. Parsing the dependency tree
3. Verifying signatures
4. Checking whether acknowledgment.uri exists
   - No → continue installation
   - Yes → call the reward plugin
5. Reward succeeds → continue installation
6. Reward fails → abort installation
7. Generating package.lock

13.2 What Core Does Not Know

Core does not know about:

- Alipay
- WeChat Pay
- Stripe
- PayPal
- GitHub Star
- Twitter Follow
- Enterprise approval workflows
- Any specific form of reward

This decoupling ensures OPPS's generality and long-term stability.

Core knows only one thing: there is a URI, call the plugin, wait for the result. Nothing else.

---

Chapter 14: Fork

14.1 Forking is Allowed

OPPS fully permits forking.

14.2 Modifications Require Re-signing

If package.toml is modified, it must be re-signed, otherwise installation fails.

14.3 Fully Forked Packages

If you fork a package with reward requirements, modify acknowledgment.uri to point to your own account, and re-sign and publish:

- The new package is your version
- The original author's rewards are unaffected
- Users can choose to trust your version

---

Chapter 15: Free

15.1 The Only Way to Be Free

There is only one way to be free: no acknowledgment.uri

15.2 Unsupported "Free" Variants

The OPPS protocol does not include the following concepts:

- Donation
- Enterprise
- Subscription
- Trial

These are all handled by acknowledgment.uri and reward plugins themselves.

Design rationale: keep the protocol minimal.

---

Chapter 16: Enterprise

16.1 Enterprises Do Not Need Special Protocols

Enterprise scenarios do not require special protocol support.

16.2 Enterprise Internal Procurement

Enterprises can use internal reward plugins:

- Call the enterprise procurement SDK
- Automatically approve through internal approval workflows
- No personal bank accounts or wallets involved

16.3 Core is Unaware

Core still does not know about the concept of "enterprise".

It just calls a plugin that happens to be developed internally by the enterprise.

---

Chapter 17: Security Model

17.1 Threat Model

| Threat | Mitigation |
|--------|------------|
| Tampering with acknowledgment.uri | Signature mechanism + hash verification |
| Tampering with source code | Signature mechanism + hash verification |
| Forging reward proof | Platform digital signature |
| Replaying reward proof | Proof includes timestamp + unique action ID |
| Bypassing reward plugin | Plugin is trusted by user; Core has no bypass |
| Malicious plugin stealing information | User chooses which plugins to trust |

17.2 Trust Model

- User trusts reward plugins (e.g., official Stripe plugin, GitHub Star verification plugin)
- User trusts the author's public key (via Web of Trust or certificate system)
- OPPS Core itself does not need to trust any third party

---

Chapter 18: Compatibility with Existing Package Managers

18.1 Compatibility Strategy

OPPS is designed to be gradually compatible with existing package managers via plugins or extensions:

| Package Manager | Compatibility Method |
|-----------------|---------------------|
| npm | Via npm install hooks or custom registry |
| pip | Via pip install --find-links or custom index |
| Cargo | Via custom registry |
| Maven | Via custom repository |
| Homebrew | Via formula extension |

18.2 Gradual Migration

- Existing packages do not need modification to be read by OPPS-compatible clients.
- Packages without acknowledgment.uri behave exactly the same.
- Authors are free to choose whether to add acknowledgment.uri.

---

Chapter 19: RFC Specification Split

It is recommended to split OPPS into a series of independent RFC documents:

| RFC Number | Title | Description |
|------------|-------|-------------|
| RFC-0001 | Package Metadata | package.toml format, required/optional field definitions |
| RFC-0002 | Repository Protocol | Upload, download, search, mirror API specification |
| RFC-0003 | Signature | Digital signature algorithm, hash rules, verification process |
| RFC-0004 | Acknowledgment Plugin API | Plugin interface definition, return values, error codes |
| RFC-0005 | Acknowledgment URI | acknowledgment.uri field definition and general requirements |
| RFC-0006 | Lock File | package.lock format, reward proof recording |
| RFC-0007 | Security Model | Threat model, trust model, best practices |
| RFC-0008 | CLI Interface | Command-line tool specification |

---

Frequently Asked Questions (FAQ)

Q1: Does this violate open source definitions?

A: No. OPPS is a package management protocol, not a license. The license can still be MIT, GPL, or Apache.

OPPS does not restrict anyone from modifying or redistributing — it only provides a way to express acknowledgment to the author at install time.

Q2: How is this different from donations?

A: Fundamentally different.

- Donations are post-hoc, voluntary, and unstable. The user uses the software, likes it, and might donate — or might not.
- OPPS is up-front, automated, and systematic. The user completes the reward action before downloading — whether paying money or giving a Star.

Donations rely on charity mentality; OPPS relies on value exchange.

Q3: What if no one pays / gives a Star?

A: That's a market choice problem, not a protocol problem.

OPPS only provides a "capability", not a "mandate". If an author believes their software deserves a certain form of reward, they can choose to require it; if users don't think it's worth it, they can choose not to use it.

In highly competitive areas (like JSON parsing), reward requirements will tend toward zero or very low. In highly specialized areas (like game engines), rewards are more readily obtained.

OPPS itself does not intervene in market pricing.

Q4: Will enterprises just fork to bypass?

A: Forking is fully compliant, but requires re-signing.

An enterprise can fork a package, remove acknowledgment.uri, re-sign, and use it internally. This is fully in line with the spirit of open source.

If an enterprise chooses to do this, it means either the package's reward requirement is unreasonable, or the enterprise is willing to bear the long-term cost of maintaining a fork.

Q5: What if the author's reward plugin stops working?

A: If the reward method specified by the author becomes unavailable (e.g., a payment platform shuts down), users can still reinstall using the existing package.lock and reward proof.

If the user needs a new version, they must contact the author or find an alternative.

Q6: Isn't this just automating guilt-tripping?

A: No. Guilt-tripping is "you should give me money/Star, even though you have no obligation."

OPPS is "if you want to download this package, you need to complete this action first." This is a clear exchange condition, not a vague moral expectation.

Users have complete freedom of choice: accept the condition and download, or reject it and do not download. There is no "guilt-tripping."

Q7: How is this different from GitCoin / Polar / OpenCollective?

A: They are donation/funding platforms (post-hoc payment).

OPPS is an install-time reward protocol (up-front reward).

Both can coexist:

- Donation platforms are for "thanking the author."
- OPPS is for "rewarding the author for obtaining software."

---

Summary

The core value of OPPS is not about "inventing a new payment system" or "overthrowing existing package managers."

Its value lies in two things:

First, decoupling "reward" from "package manager," making reward a plugin-based capability independent of any platform.

Through an optional acknowledgment.uri field and a unified plugin interface, OPPS gives every developer the ability to receive reward — whether money, attention, or gratitude.

Second, redefining "payment."

In OPPS, payment is not just money. It is any form of reward a user gives to an author when obtaining software.

Money is reward. A Star is reward. Following is reward. Saying thank you is reward.

This gives open source authors, for the first time, a standardized way to have their labor seen, recognized, and rewarded. The specific form is decided by the author, executed by the plugin, and orchestrated by Core.

Whether you are an independent author hoping to make a living from code, a developer who just wants to be known by more peers, or a hacker who wants nothing and purely loves sharing — OPPS offers the same respect to all:

The right to choose.

OPPS is not the end. It is only the beginning.

---

- Version: 1.1.0
- Status: Draft
- Author: live.aileen / Bai Qianxi
- Updated: 2026-07-03
- Link: https://github.com/OPPS/OPPS
- Email: live.aileen@outlook.com
- QQ: 3142116244
- QQ Group: 1046801466
---
