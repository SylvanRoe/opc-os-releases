# OPC OS 交付清单

> 适用版本：v0.6.0（2026-08-28，对齐 v0.4 非 docker 原生进程形态 + v0.5.1 监控/告警/守护组件合入 + v0.6 升级机制）。本清单用于部署包交付核对与验收对照。
> 产品名口径：**OPC OS**（AI 原生团队的治理操作系统），对外文案一律用此名。
> v0.3 变更：v0.2 为过时 docker 形态（compose/镜像/卷），v0.4 起部署形态已改为**非 docker 原生进程**
> （systemd 守护，无 systemd 退化 nohup）——本清单逐条对齐部署包实际，每条 = 一个可执行验证动作。
> v0.3.1 变更（P2）：5 个 unit 模板统一固化 `EnvironmentFile` 约束（`@@ENV_FILE@@` 变量渲染，`--env-file` 可覆盖），
> 新增验收点 17 与 installation-guide §8.8/§11.6 核查命令。
> v0.3.2 变更（Gavin 0827「公司即产品」拍板）：生产验证的监控/告警/守护组件合入——
> 告警总线（opc-alert-bus.py + alerts_bus_monitor.py）、投递看门狗（telegram_delivery_watchdog.py）、
> 代理守护（singbox-watchdog.sh + systemd/sing-box.service.in）、总线自测（test_alert_bus.py），
> 新增验收点 18-20 与 installation-guide §11.5。system-monitor.py（降载感知）验收点 21 当时因 P0 复验未过暂缓合入（2026-08-28 补记：已合入，见下）。
> v0.3.2 补记（2026-08-28，卡 t_eec83597）：验收点 21 转正——system-monitor.py（降载感知/自愈）随 P0 复验 PASS（t_d1b7cc94）合入 scripts/（commit a0039b9），env 全参数化（OPC_*，默认值生产路径兜底）+ 自测 scripts/.system-health/test_degrade_perception.py 19/19 进包。
> v0.4 变更（Gavin 0827 升级机制指令）：升级机制合入——版本检查/通知（opc-os-check-update.py）、安全升级
> （opc-os-upgrade.sh，备份先行→增量→冒烟→失败自动回滚）、备份集扩展 4→7 文件、VERSION 单一来源；
> 新增验收点 22-25 与 installation-guide §9 升级机制章节。tag 发布规则待 G 卡 G-2026-08-27-002 拍板。

---

## 1. 交付物清单

### 1.1 服务（5 个 systemd unit，原生进程）

| unit | 进程 | 端口 | 说明 |
|---|---|---|---|
| `opcos-gateway.service` | `node server.js`（llm-gateway/） | 18080（仅 127.0.0.1） | BYOK 透明网关（熔断/预算/多上游路由，统一模型名 `hermes-opc-v1`） |
| `opcos-api.service` | `node index.cjs`（opc-api/） | 3033 | 透明办公室/工作台/治理 API |
| `opcos-web.service` | `node scripts/web-serve.js`（零依赖静态服务器） | 8081 | 透明办公室前端（/opc/* 页面 + /opc/data/* 数据） |
| `opcos-collector.service` | `python3 scripts/opc-bus.py`（数据总线守护）+ 宿主机 crontab | — | 1s 轮询 kanban.db → 秒级刷新 status.json kanban 段 |
| `opcos-dispatcher.service` | `python3 -m kanban_dispatcher.cli`（fork venv） | — | 弹性调度（60s tick，容量自动推导） |

- 模板：`systemd/opcos-*.service.in`（install-services.sh 渲染 `@@VAR@@` → `/etc/systemd/system/opcos-*.service`）；
- crontab（install-services.sh 安装）：`*/1` opc_collect.py + token_stats.py、`*/5` llm_gateway_sync.py；
- 无 systemd 环境（精简 LXC）自动退化 nohup+pidfile（`install-services.sh start/stop/status`）。

### 1.1b 可选守护：sing-box 代理（v0.5.1 合入）

| unit | 进程 | 形态 | 说明 |
|---|---|---|---|
| `sing-box.service.in` | `sing-box run -c ~/.config/sing-box/config.json`（用户代理） | systemd **user** unit（`~/.config/systemd/user/`） | Telegram 等外部投递链路的关键底层（08-27 生产事故归因：代理节点抖动 → 投递超时）。`Restart=always` + `StartLimitIntervalSec=0` 守护化。`%h` 自动解析部署用户 home，天然可移植 |

- 安装（手动三行，install-services.sh 自动接入为 v0.6 迭代项）：`cp systemd/sing-box.service.in ~/.config/systemd/user/sing-box.service && systemctl --user daemon-reload && systemctl --user enable --now sing-box`
- 前提：`~/.local/bin/sing-box` 二进制 + `~/.config/sing-box/config.json` 由部署者自备（代理配置不随包分发）；
- 无 systemd 环境退化：由 `singbox-watchdog.sh` nohup 守护覆盖重启。

### 1.2 install 三件套

| 脚本 | 作用 |
|---|---|
| `scripts/opc_init.py` | 参数化生成客户团队（`--template software/manufacturing/retail/aigc`、`--name`、`--gateway`、`--out`）：config/ + hermes-home profiles 骨架 + state.json + kanban.db + init-state.json + llm.env(600) + init-report |
| `scripts/install-hermes-worker.sh` | 客户 Hermes 运行时：装 fork（锁 tag v0.20.5-opc.1，`--source` 本地源码或 GITHUB_TOKEN 私有仓）+ 每 profile 官方 `hermes config set` 写网关配置 + key 只进 llm.env(600) + 幂等 + 完成校验 |
| `scripts/install-services.sh` | 一键安装/启停 5 服务：`install/start/stop/status/restart/uninstall`；`install` 幂等（config.native.json 路径重写 + dispatcher.yaml + secrets.env(600) + env.sh(600) + crontab + systemd units） |

### 1.3 数据与运维脚本

| 文件 | 说明 |
|---|---|
| `scripts/opc_collect.py` | 状态采集（→ status.json，段级写协调） |
| `scripts/token_stats.py` | token 归因（→ token-stats.json：用量/成本/任务/通过率） |
| `scripts/opc-bus.py` | 数据总线守护（1s 增量轮询 kanban.db task_events → status.json kanban 段，水位线 bus-state.json，原子写） |
| `scripts/llm_gateway_sync.py` | 网关状态同步（→ llm-gateway.json，*/5） |
| `scripts/web-serve.js` | 零依赖静态服务器（clean-URL 白名单 + /opc/data 直连 opc-init 输出目录） |
| `scripts/opc_kanban_reopen.py` | kanban 重开 runner（opc-api 调用） |
| `scripts/backup.sh` | 数据备份（v0.6 扩展 7 文件集：核心 kanban.db/status.json/token-stats.json/state.json 缺一即失败中止 + 配置 gateway.config.json/bus-state.json/govern-state.json 缺一警告继续 → backups/<日期>/，默认保留 14 份；`--keep/--out/--data-dir`；env OPC_DATA_DIR/OPC_GATEWAY_CONFIG） |
| `scripts/opc-os-check-update.py` | **版本检查/通知**（v0.6 合入）：读 VERSION vs 远端最新（GitHub Releases → git ls-remote），semver 比较；有新版 → alert-bus（type=upgrade-available）+ `~/.opc-os/update-available.json` 双落点；离线静默退出 0；`--json/--force`；install-services crontab 每日定时（OPC_UPDATE_CHECK_INTERVAL 可配） |
| `scripts/opc-os-upgrade.sh` | **安全升级**（v0.6 合入）：阶段化流程（前置检查→自动备份→目标解析→增量/全量判定→冒烟→失败自动回滚→upgrade.log 留痕）；`--to vX.Y.Z/--dry-run/--rollback/--force`；主版本变化拒绝自动升级输出全量指引；env 全参数化（OPC_OS_REPO/OPC_OS_STATE_DIR/OPC_DATA_DIR/OPC_GATEWAY_CONFIG/OPC_SMOKE_PORT） |
| `scripts/smoke_test.sh` | 本地验收冒烟（**37 项**，配 WORKBENCH_API 时 38 项；进程/HTTP/clean-url/workbench/数据/总线/脱敏红线） |
| `scripts/opc-os-doctor.py` | 自助诊断（10 组检查 / --json / 退出码 0-1-2 / 修复建议；**随 P0-3 交付合入**，见 §2 验收点 7） |
| `scripts/opc-alert-bus.py` | **告警总线**（v0.5.1 合入）：监控/看门狗告警统一入 `alerts/bus.jsonl`（O_APPEND 原子写，schema 定稿 9 字段），CLI `append/read_new/mark_routed/mark_resolved/count_new/list`，P2 直接 resolved 留痕；路径 `OPC_ALERT_DIR` env 可覆盖（默认 `/home/gavin/.hermes/opc/alerts`，生产兜底） |
| `scripts/alerts_bus_monitor.py` | **告警路由 monitor**（v0.5.1 合入）：告警总线 → 路由 agent 的 monitor_script（5min tick，输出稳定摘要触发路由 agent 分级建卡；未处理 age>2h 打 `[STALE>2h]`；18:50 汇总窗口） |
| `scripts/telegram_delivery_watchdog.py` | **TG 投递看门狗**（v0.5.1 合入）：日志滑动窗口错误统计（ConnectError/Timed out/blocked_config）+ jobs.json blocked_config 扫描；no_agent cron 静默契约，异常才输出 → 自动建卡闭环（经告警总线）；`WD_*` env 可覆盖阈值/路径 |
| `scripts/singbox-watchdog.sh` | **代理自愈看门狗**（v0.5.1 合入）：阶梯自愈（L1 SIGHUP → L2 restart → L3 熔断 30min/≥3 次）+ v2.1 持续窗口（telegram 非 connected ≥5min 才重启 gateway，防瞬态误杀）+ 运行中任务保护；`SCRIPT_DIR/LOG_FILE/GW_STATE_JSON/EXECUTIONS_DB` env 可覆盖 |
| `scripts/test_alert_bus.py` | 告警总线单元自测（24 断言：append 原子性并发不丢行 / P2 resolved / read_new 过滤 / mark 状态迁移 / fingerprint 公式 / evidence 截断 / CLI 冒烟；临时 ALERT_DIR 零污染生产） |
| `scripts/test_upgrade_mechanism.py` | **升级机制自测**（v0.6 合入，46 项）：mock 远端零污染——check-update 间隔跳过/离线静默/有新版双落点/无新版清残留、upgrade 主版本拒绝/--dry-run、backup 7 文件集核心缺一中止/配置缺一警告 |

### 1.4 配置与站点

| 文件 | 说明 |
|---|---|
| `config/config.json` | opc-init 产物模板（profiles/cron_member/名册/角色/paths/running_since/bus；容器视角 /data 路径，install 时重写为 config.native.json） |
| `config/dispatcher.yaml` | dispatcher 配置模板（hermes_home/boards/interval/stale_timeout） |
| `systemd/opcos-*.service.in` | 5 个 unit 模板（@@OUT@@/@@REPO@@/@@NODE@@/@@PYTHON@@/@@USER@@/端口/容量/**@@ENV_FILE@@** env 渲染；5 个全部含 `EnvironmentFile` 实线，install 自动解析注入面） |
| `web/opc/*` | 透明办公室（index.html）+ 治理仪表盘（governance.html）+ 工作台（workbench.html，OPC_WORKBENCH_API 注入点） |
| `web/locales/*.json` | 12 语言包（ar/de/en/es/fr/ja/ko/pl/pt/ru/tr/zh） |
| `vendor/kanban-dispatcher/` | dispatcher 源码（锁 commit，离线可用） |

### 1.5 文档

| 文件 | 说明 |
|---|---|
| `README.md` | 英文主文档（是什么/快速开始/配置/数据自持/升级/FAQ） |
| `README_CN.md` | 中文文档（v0.5 同步：非 docker 原生进程形态 + doctor/冒烟/守护） |
| `deploy-guide.md` | 部署手册（前置/分步部署/验证清单/升级回退/故障排查） |
| `docs/installation-guide.md` | 客户形态从零安装手册 v0.6（§9 升级机制/§10 排障/§11 监控/§12 恢复 四章齐） |
| `delivery-checklist.md` | 本清单（v0.4） |

### 1.6 版本

- **版本单一来源 = 仓根 `VERSION` 文件**（当前 `0.6.0`）：`opc_init.py --version` / check-update / upgrade 全部读取；发布 = VERSION + CHANGELOG + git tag 三处对齐；
- 部署包按 git tag 发布（当前最新 git tag v0.3.5，v0.6.0 tag 待 G 卡 G-2026-08-27-002 拍板发布规则后补齐）；
- 底座 fork 锁 tag `v0.20.5-opc.1`（install-hermes-worker.sh `--tag` 换版本升级）。

---

## 2. 验收标准对照（每条 = 一个可执行验证动作）

| # | 验收点 | 标准 | 验证方式 |
|---|---|---|---|
| 1 | 全新机器从零部署 | 全新 Ubuntu 24.04（≥2 核 4G）按 installation-guide v0.5 逐步执行：环境自举 → 获取部署包 → opc-init → install-hermes-worker → install-services install 全通 | 手册 §3-§7 逐节照抄，命令零修改 |
| 2 | 5 服务在线 | `systemctl is-active opcos-gateway opcos-api opcos-web opcos-collector opcos-dispatcher` | 全部 active；无 systemd 时 `install-services.sh status` 全部运行中 |
| 3 | 一键冒烟 | `scripts/smoke_test.sh 8081` | **37/37 ALL GREEN**（配 WORKBENCH_API 时 38/38） |
| 4 | 网关链路 | `/v1/models` 200 且仅 `hermes-opc-v1`；真实 chat 调用 200（路由上游）；admin 面板 `/admin` 可开、API 无 token 401 | 手册 §8.1/§7.4 命令照抄 |
| 5 | 成员走网关 | `source <out>/env.sh && hermes -p <成员> chat -q "hi"` 经网关成功 | 手册 §8.2；会话落盘 profiles/<id>/state.db |
| 6 | kanban 全流程 | 建卡 → dispatcher 60s tick 派活 → worker 完成 → 事件留痕 | 手册 §8.3；`journalctl -u opcos-dispatcher` 见 spawned=N |
| 7 | doctor 可诊断 | `python3 scripts/opc-os-doctor.py` 10 组检查逐组可跑；全绿退出码 0；`--json` 输出合法 JSON；造故障（停 1 服务）退出码 1 + 定位准确 + 修复建议可执行；全程无明文凭据输出 | **doctor 已随 P0-3 交付合入部署包**（scripts/opc-os-doctor.py） |
| 8 | 数据真实流动 | 仪表盘按成员展示 token/成本/任务数/通过率，来自 token-stats + kanban 非写死 | `curl /opc/data/token-stats.json` 查字段；`smoke_test.sh` 数据段 |
| 9 | 数据总线 | opc-bus 存活；status.json 含 `sources.kanban`（ts/watermark/events）；水位线 bus-state.json 经 web 可读 | `pgrep -af opc-bus.py`；`smoke_test.sh` 总线段 |
| 10 | crontab 采集 | `*/1` opc_collect + token_stats、`*/5` llm_gateway_sync 两条在 | `crontab -l \| grep OPC-OS`；`<out>/logs/*.log` 有刷新 |
| 11 | 参数化 | 改 config.json 的 profiles/路径后实例正常，脚本零本机硬编码；`--web-port/--api-port/--gateway-port/--tier/--mem-mb/--cpus` 生效 | 手册 §7.1 参数表；改端口重装验证 |
| 12 | 备份/恢复（v0.6 扩展 7 文件） | `backup.sh --data-dir <out>` 产出 7 文件到 backups/<日期>/（核心 4 + 配置 3）；核心缺 1 文件时失败中止并清理本次部分复制；配置类缺 1 文件时警告继续退出 0；恢复步骤照 §12.2 数据一致 | 手册 §12；「备份 → 删数据 → 恢复 → smoke 绿」演练 |
| 13 | 增量升级/回退 | 备份先行 → git pull/checkout tag → install 重跑 → smoke 绿；回退 checkout 旧 tag + install 重跑数据不动 | 手册 §9；升级演练留痕 |
| 14 | 红线 | 无明文 key 落代码/配置/文档（一律占位符）；llm.env/secrets.env 权限 600；零真实团队身份；运行时零外连 | 手册 §14 红线复核命令；`grep sk-` 零命中；`stat -c %a` = 600 |
| 15 | 文档一致 | README_CN / installation-guide / deploy-guide 描述与部署行为一致，照文档能复现 | 新手按 installation-guide v0.6 §3-§8 逐步复现；grep docker 业务段落零命中 |
| 16 | 交付物/git | 本清单 §1 全部在位，git 工作区干净 | `git status` clean；文件逐一核对 |
| 17 | EnvironmentFile 约束（P2 固化） | 5 个 unit 模板全部含 `EnvironmentFile=@@ENV_FILE@@`；install 渲染后实机 unit 各 1 行 `EnvironmentFile=` 且指向存在的 env 文件（600）；`--env-file` 可覆盖注入面 | 手册 §8.8 命令照抄：`grep -c '^EnvironmentFile=' /etc/systemd/system/opcos-*.service` 应 5×1；`stat -c %a` = 600；doctor 组 3 模板检查绿 |
| 18 | 告警总线可用（v0.5.1） | `python3 scripts/opc-alert-bus.py` 在临时 `OPC_ALERT_DIR` 下 append → read_new → mark_resolved 全链路工作；`scripts/test_alert_bus.py` 全绿退出 0 | 手册 §11.5.1 命令照抄（临时目录零污染）；`test_alert_bus.py` 输出「24 通过 / 0 失败」 |
| 19 | 投递看门狗部署（v0.5.1） | telegram_delivery_watchdog.py 按 no_agent cron 30min 挂载；正常态 stdout 空（静默）、异常态输出非空且自动入告警总线 | 手册 §11.5.2：跑一次 `python3 scripts/telegram_delivery_watchdog.py`（正常态无输出）；`crontab -l` 见 watchdog 条目 |
| 20 | 代理守护（v0.5.1） | `systemctl --user is-active sing-box` = active；`systemctl --user show sing-box -p Restart` = always；无 sing-box 环境标注跳过 | 手册 §11.5.3 命令照抄；watchdog 输出含 `proxy OK` |
| 21 | 降载感知合入 | system-monitor.py（降载登记 + P2 受控降载留痕 + 端口剔除）已随 P0 复验 PASS 合入部署包 scripts/system-monitor.py，env 全参数化（OPC_* 覆盖，默认值生产路径兜底零行为变化） | 自测 scripts/.system-health/test_degrade_perception.py 19/19 全绿；合入 commit a0039b9（卡 t_eec83597，2026-08-28） |
| 22 | 版本单一来源（v0.6） | 仓根 `VERSION` 文件存在且内容 `0.6.0`；`python3 scripts/opc_init.py --version` 输出与 VERSION 一致；opc-os-check-update.py / opc-os-upgrade.sh 读 VERSION 文件（改 VERSION 后两脚本行为随之变化） | `cat VERSION`；`python3 scripts/opc_init.py --version` 比对；§9.1 | 
| 23 | 版本检查/通知（v0.6） | `python3 scripts/opc-os-check-update.py --force --json` 输出合法 JSON（含 current/update_available）；`OPC_UPDATE_CHECK_URL` 指向本地 mock 新版时 alert-bus 出现 upgrade-available（P2）且 `~/.opc-os/update-available.json` 双落点；远端全失败（URL 指向不存在）静默退出 0；`crontab -l` 有每日 check-update 行（默认 04:30，OPC_UPDATE_CHECK_INTERVAL 可配） | 手册 §9.2 命令照抄；`test_upgrade_mechanism.py` 覆盖；`crontab -l \\| grep check-update` |
| 24 | 安全升级/回滚（v0.6） | `bash scripts/opc-os-upgrade.sh --dry-run` 输出前置检查 + 目标解析不实际变更；`--to` 主版本不同（如 1.0.0）拒绝自动升级输出全量指引退出非 0；`--rollback` 回到上一可用 tag 并恢复备份数据；`~/.opc-os/upgrade.log` 有 {ts, from, to, result} 留痕 | 手册 §9.3/§9.7 命令照抄；`test_upgrade_mechanism.py` 46/46 覆盖（mock 远端零污染） |
| 25 | 升级自测进包（v0.6） | `python3 scripts/test_upgrade_mechanism.py` 全绿退出 0（46 项：check-update 间隔跳过/离线静默/双落点/清残留、upgrade 主版本拒绝/--dry-run、backup 7 文件集） | 自测输出「46 通过 / 0 失败」；临时目录零污染生产 |
| 26 | 网关配置并发写保护（v0.7） | llm-gateway 内置写锁+版本戳CAS+终态校验：库内 `grep writeFileSync(gateway.config.json)` 只剩 `lib/admin.js` 原子写；`npm test` 全绿；并发写模拟两写者一成一拒(409)；`opc-gw-config show` 读当前方向+directive+config_version | `docs/gateway-config-change-sop.md` 命令照抄；连接真实网关后 CLI `show` 输出 current_config 断言 |

---

## 3. 支持期说明（红线）

- **不承诺 SLA**：交付后的问题响应为 best-effort，无响应时限、无可用性承诺。
- **支持范围**：部署包本身的缺陷（脚本 / 编排 / 文档与实际行为不符）。
- **不在范围内**：使用方自改配置 / 自接真实数据后的环境问题；宿主机系统 / 网络环境
  本身的问题；新增功能需求（另立需求，另行评估）。
- **版本政策**：底座 fork 每周 merge upstream；部署包按 git tag 发布，建议跟随 tag 升级（installation-guide §9）。
- **变现口径**：本交付不含任何定价 / 收款 / 付费承诺；商业化事项一律由产品所有者决策，
  不在本清单范围内。

---

## 4. 交接核对签字项（人工）

- [ ] 全新机器按 installation-guide v0.6 从零部署成功（§2 验收点 1）
- [ ] `smoke_test.sh` 37/37 全绿（§2 验收点 3）
- [ ] doctor 10 组检查全绿退出 0（随 P0-3 交付；§2 验收点 7）
- [ ] 治理仪表盘 9 行成员数据可见且非写死（§2 验收点 8）
- [ ] 备份脚本导出 7 文件（核心 4 + 配置 3）与数据目录一致、恢复演练通过（§2 验收点 12）
- [ ] 文档四份（README_CN / installation-guide v0.6 / deploy-guide / 本清单）与实测行为一致
- [ ] EnvironmentFile 约束核查通过：5 个 unit 各 1 行 `EnvironmentFile=` 指向 600 env 文件（§2 验收点 17）
- [ ] 告警总线 + 看门狗 + 代理守护按 §2 验收点 18-20 实测通过
- [x] system-monitor 降载感知随 P0 验收合入（§2 验收点 21，commit a0039b9）
- [ ] 升级机制按 §2 验收点 22-25 实测通过（VERSION 单一来源 / check-update 通知 / upgrade 升级回滚 / 自测 46 项）
- [ ] git 工作区干净、发布 tag 存在（§2 验收点 16）