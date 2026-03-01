---
summary: "CLI 参考：`openclaw secrets`（reload、audit、configure、apply）"
read_when:
  - 需要在运行时重新解析 secret 引用
  - 审计明文残留和未解析的引用
  - 配置 SecretRef 并执行单向清除变更
title: "secrets"
x-i18n:
  source_path: docs/cli/secrets.md
  source_hash: f12140702d25bd4dd17582bd1b6c00e065ecf95e21038dd2caf4828cfd4b6071
  workflow: manual
  translator: xingzihai
---

# `openclaw secrets`

使用 `openclaw secrets` 将凭证从明文迁移到 SecretRef，并保持活跃的 secrets 运行时处于健康状态。

各子命令职责：

- `reload`：通过 gateway RPC（`secrets.reload`）重新解析引用，仅在完全成功时原子替换运行时快照（不写入配置文件）。
- `audit`：对配置、auth 存储及旧版残留（`.env`、`auth.json`）进行只读扫描，检查明文、未解析引用和优先级漂移。
- `configure`：交互式规划工具，用于提供商设置、目标映射和预检（需要 TTY）。
- `apply`：执行已保存的计划（`--dry-run` 仅做验证），然后清除已迁移的明文残留。

推荐的操作循环：

```bash
openclaw secrets audit --check
openclaw secrets configure
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json
openclaw secrets audit --check
openclaw secrets reload
```

CI/门控的退出码说明：

- `audit --check` 发现问题时返回 `1`，引用未解析时返回 `2`。

相关文档：

- Secrets 指南：[Secrets Management](/gateway/secrets)
- 安全指南：[Security](/gateway/security)

## 重载运行时快照

重新解析 secret 引用，并原子替换运行时快照。

```bash
openclaw secrets reload
openclaw secrets reload --json
```

说明：

- 使用 gateway RPC 方法 `secrets.reload`。
- 若解析失败，gateway 保留上一个已知正常的快照并返回错误（不会部分激活）。
- JSON 响应中包含 `warningCount`。

## 审计

扫描 OpenClaw 状态，检查：

- 明文 secret 存储
- 未解析的引用
- 优先级漂移（`auth-profiles` 遮蔽了配置中的引用）
- 旧版残留（`auth.json`、OAuth 范围外说明）

```bash
openclaw secrets audit
openclaw secrets audit --check
openclaw secrets audit --json
```

退出行为：

- `--check` 在有发现时以非零退出。
- 未解析的引用以更高优先级的非零退出码退出。

报告关键字段：

- `status`：`clean | findings | unresolved`
- `summary`：`plaintextCount`、`unresolvedRefCount`、`shadowedRefCount`、`legacyResidueCount`
- 发现代码：
  - `PLAINTEXT_FOUND`
  - `REF_UNRESOLVED`
  - `REF_SHADOWED`
  - `LEGACY_RESIDUE`

## 交互式配置助手

以交互方式构建提供商与 SecretRef 变更，执行预检，并可选择直接应用：

```bash
openclaw secrets configure
openclaw secrets configure --plan-out /tmp/openclaw-secrets-plan.json
openclaw secrets configure --apply --yes
openclaw secrets configure --providers-only
openclaw secrets configure --skip-provider-setup
openclaw secrets configure --json
```

流程：

- 首先配置提供商（对 `secrets.providers` 别名执行添加/编辑/删除）。
- 然后进行凭证映射（选择字段并分配 `{source, provider, id}` 引用）。
- 最后执行预检，并可选择应用。

参数说明：

- `--providers-only`：仅配置 `secrets.providers`，跳过凭证映射。
- `--skip-provider-setup`：跳过提供商设置，直接将凭证映射到已有提供商。

注意事项：

- 需要交互式 TTY。
- `--providers-only` 与 `--skip-provider-setup` 不可同时使用。
- `configure` 的目标是 `openclaw.json` 中所有包含 secret 的字段。
- 请将所有计划迁移的 secret 字段一并纳入（例如同时包含 `models.providers.*.apiKey` 和 `skills.entries.*.apiKey`），以便审计后达到干净状态。
- 应用前会执行预检解析。
- 生成的计划默认开启清除选项（`scrubEnv`、`scrubAuthProfilesForProviderTargets`、`scrubLegacyAuthJson` 均已启用）。
- 应用路径对已迁移的明文值是单向的，不可逆。
- 若不带 `--apply`，CLI 仍会在预检后询问 `是否立即应用此计划？`。
- 带 `--apply`（且不带 `--yes`）时，CLI 会额外询问一次不可逆迁移确认。

Exec 提供商安全说明：

- Homebrew 安装通常会在 `/opt/homebrew/bin/*` 下暴露符号链接的二进制文件。
- 仅在需要时为可信的包管理器路径设置 `allowSymlinkCommand: true`，并配合 `trustedDirs`（例如 `["/opt/homebrew"]`）一起使用。

## 应用已保存的计划

应用或预检之前生成的计划：

```bash
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --json
```

计划合约详情（允许的目标路径、验证规则和失败语义）：

- [Secrets Apply Plan Contract](/gateway/secrets-plan-contract)

`apply` 可能更新的内容：

- `openclaw.json`（SecretRef 目标 + 提供商的增删改）
- `auth-profiles.json`（提供商目标清除）
- 旧版 `auth.json` 残留
- `~/.openclaw/.env` 中已迁移值对应的已知 secret key

## 为何不提供回滚备份

`secrets apply` 有意不写入包含旧明文值的回滚备份。

安全性来自严格的预检 + 近似原子的应用流程，以及失败时的尽力内存恢复。

## 示例

```bash
# 先审计，再配置，最后确认状态干净：
openclaw secrets audit --check
openclaw secrets configure
openclaw secrets audit --check
```

若部分迁移后 `audit --check` 仍报告明文发现，请确认也迁移了 skill key（`skills.entries.*.apiKey`）以及其他所有报告的目标路径。
