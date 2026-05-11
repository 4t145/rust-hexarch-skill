# rust-hexagonal-architecture-skill

一个 Claude Code skill，用于在编写 Rust 后端/HTTP 服务代码时自动应用六边形架构（ports & adapters）规范。

## 来源致谢

本 skill 的全部内容提炼自以下第三方资料，版权归原作者所有：

- **原文教程**：[Master Hexagonal Architecture in Rust](https://www.howtocodeit.com/guides/master-hexagonal-architecture-in-rust) — howtocodeit.com
- **官方参考实现**：[github.com/howtocodeit/hexarch](https://github.com/howtocodeit/hexarch) —— 本 skill 中的代码样板、命名约定、目录结构以 [`3-simple-service`](https://github.com/howtocodeit/hexarch/tree/3-simple-service) 分支为基准
- **相关前置阅读**：[The Ultimate Guide to Rust Newtypes](https://www.howtocodeit.com/articles/ultimate-guide-rust-newtypes)

本仓库不是原作者发布的官方 skill，而是社区对原文的再整理，意图是让 AI 助手（Claude Code）在日常写代码时能直接把这套架构作为默认规范落地。如有问题，请先以原文和官方仓库为准。

## 它做什么

- 当你让 Claude 写 axum/actix 的 HTTP 服务、设计 domain 层、加 repository/service trait 时自动触发
- 强制 10 条硬性规则（domain 不 import infra、错误必有 `Unknown(anyhow::Error)` 兜底、newtype 校验、request ≠ entity 等）
- 提供可直接复制的目录骨架和代码模板
- 用 Before/After 反例提示常见误区（比如把 port 命名成 entity 而不是 bounded context）

## 它不做什么

Skill 的 `description` 明确排除以下场景，会主动告诉你不适用：

- 单人原型、脚本、CLI 工具
- 业务逻辑极少的 thin CRUD 代理
- 性能敏感的热路径
- 库 crate（不该把架构强加给调用方）

## 目录结构

```
rust-hexagonal-architecture-skill/
├── README.md                         # 本文件
├── SKILL.md                          # skill 入口：触发描述 + 核心规则 + 自查清单
├── anti-patterns.md                  # 12 组 Before/After 反例
├── reference/                        # 按需加载的细节文档
│   ├── structure.md                  # Cargo.toml 和完整目录布局
│   ├── ports.md                      # trait 签名和命名（bounded context vs entity）
│   ├── errors.md                     # 三层错误体系 + From 映射
│   ├── handlers.md                   # HTTP handler 三步模式
│   └── testing.md                    # 在哪一层 mock 什么 port
└── templates/                        # 可复制的代码骨架
    ├── domain_module.rs.tmpl         # 新 bounded context 骨架
    ├── adapter_sqlite.rs.tmpl        # sqlx adapter 骨架
    └── main.rs.tmpl                  # 仅 bootstrap 的 main
```

## 安装

Claude Code 会扫描 `~/.claude/skills/` 和 `<project>/.claude/skills/` 两个目录下包含 `SKILL.md` 的子目录并自动注册。根据你的使用场景选一种方式即可。

### 方式一：用户级 skill（所有项目生效，推荐个人使用）

```bash
git clone https://github.com/4t145/rust-hexarch-skill.git \
    ~/.claude/skills/rust-hexagonal-architecture
```

升级：

```bash
cd ~/.claude/skills/rust-hexagonal-architecture && git pull
```

### 方式二：项目级 skill（只在某个 Rust 项目生效）

```bash
cd <your-project>
git clone https://github.com/4t145/rust-hexarch-skill.git \
    .claude/skills/rust-hexagonal-architecture
echo ".claude/skills/rust-hexagonal-architecture/" >> .gitignore
```

### 方式三：作为 submodule 跟随项目（团队共享同一版本，推荐团队使用）

```bash
cd <your-project>
git submodule add https://github.com/4t145/rust-hexarch-skill.git \
    .claude/skills/rust-hexagonal-architecture
git commit -m "add rust hexagonal architecture skill"
```

队友拉代码：

```bash
git clone --recurse-submodules <your-project>
# 或已 clone 过：
git submodule update --init
```

升级到 skill 最新版：

```bash
cd .claude/skills/rust-hexagonal-architecture && git pull origin main
cd - && git add .claude/skills/rust-hexagonal-architecture && git commit -m "bump skill"
```

> **目录命名提醒**：skill 所在目录名（`rust-hexagonal-architecture`）建议和 `SKILL.md` 里的 `name` 字段保持一致，方便排查。仓库名（`rust-hexarch-skill`）只是 GitHub 上的标识，两者不必相同。

## 免责声明

- 本仓库内容为原文和官方实现的再整理，不代表原作者立场
- 技术细节以原文和 [hexarch 仓库](https://github.com/howtocodeit/hexarch) 的最新状态为准；如果你发现 skill 与官方实现有出入，请以官方为准并提 issue
- Rust 生态变化较快（`async fn in trait`、`return-type notation` 等语言特性持续演进），本 skill 的某些代码样板可能随时间过时
