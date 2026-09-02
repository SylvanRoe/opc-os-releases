# OPC OS Release 仓库

> 官方安装包发布仓库：仅含各版本安装包（tar.gz）与成员运行时（Hermes Agent fork）源码包，不含内部源码。

## 当前版本
- **v0.8.8**（最新发布）：安装包 `opc-os-v0.8.8.tar.gz`。
- 请在右侧 **Releases** 面板下载，安装步骤见包内 `docs/installation-guide.md`（从零到可用完整手册）。

## 下载说明
- 请使用 GitHub **Release Asset** 下载（经 CDN，完整可靠）；**不可用** raw.githubusercontent.com 直链（会被截断，无法解压）。
- 下载后请比对 release 中的校验值（`md5sum opc-os-v0.8.8.tar.gz`）。
- 源码不在此仓库公开发布。

## 成员运行时（Hermes Agent fork）
- 「AI 员工」运行在 OPC 定制版 Hermes Agent 之上（含 kanban task_kind/reopen/decompose、mcp TTL、SMTP TLS 修复等定制层）。
- 安装时 **无需手动下载**：`install-hermes-worker.sh` 自动从本仓库 Releases 获取锁定 tag 的 fork 源码包
  （`hermes-agent-fork-v0.20.5-opc.2.tar.gz`，免 token），详见包内 `docs/installation-guide.md` §4.2/§6.2。
