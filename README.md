# OPC OS

**OPC OS = AI 原生团队的治理操作系统** —— 管理 AI 成员团队的角色 / 任务 / 费用 / 用量 / 工作记录 / 验收。

[English README](README.md) · [部署手册](deploy-guide.md) · [交付清单](delivery-checklist.md) · [安装文档](docs/installation-guide.md) · [时钟服务](docs/clock-service.md)

> 适用版本：v0.6.0（客户形态 · **非 docker 原生进程**，2026-08-28 与 installation-guide v0.6.0 同步）。
> 完整从零安装手册见 [docs/installation-guide.md](docs/installation-guide.md)。

## OPC OS 是什么？

市面上的「AI 运营」工具大多在管**人怎么用 AI**：员工席位、token 预算、prompt 日志。
OPC OS 从相反的前提出发：**AI 智能体本身就是团队成员**。一人公司（或任何团队）可以
让一整个 AI 组织跑起来——CEO、开发、运维、营销、客服、财务、情报、测试、设计——
OPC OS 就是治理这个组织的系统层：

- **他们是谁** —— 有名有姓有角色的成员名册与 profile
- **他们在干什么** —— kanban 驱动的任务全生命周期（派活 → 领活 → 干活 → 验收 → 完成），事件全程留痕
- **他们花了多少** —— 按成员归因的 token 用量与成本汇总
- **活干得行不行** —— 验收 / 测试结论与按成员的通过率
- **他们「坐」在哪** —— 透明办公室页面：团队状态、实时活动流、任务看板，任何人可见

完全自托管，全部数据落在你自己的**宿主机目录**（`<opc-init 输出目录>`）里，不依赖 docker / 镜像 / 外部服务。

## 快速开始（三步）

依赖只有 node ≥ 20 + python3.10+（含 uv）。详细步骤见 installation-guide v0.5：

```bash
# ① 生成客户团队（参数化，非交互）
python3 scripts/opc_init.py --template software --name "<客户公司名>" --gateway --out ~/opc-test

# ② 安装 Hermes 团队运行时（fork 底座 + 每成员 profile 走网关）
bash scripts/install-hermes-worker.sh \
  --source ~/hermes-agent-src \
  --hermes-home ~/opc-test/hermes-home \
  --gateway-url http://127.0.0.1:18080/v1

# ③ 起栈（5 服务 systemd 守护；无 systemd 自动退化 nohup）
scripts/install-services.sh install --out ~/opc-test
```

> 环境变量注入：5 个 `opcos-*` unit 全部经 systemd `EnvironmentFile` 注入 token（默认 `<out>/secrets.env`，
> 可用 `--env-file <路径>` 覆盖，如生产 `~/.hermes/.env`）——裸 node/python 守护不自行加载 env，
> 注入面缺失即成员对话 401。核查命令见 `docs/installation-guide.md` §8.8。

访问：

- 透明办公室（虚拟办公室 / 活动流 / 任务看板）：<http://localhost:8081/opc/>
- 治理仪表盘（成员 token / 成本 / 任务数 / 通过率）：<http://localhost:8081/opc/governance.html>
- 健康检查：<http://localhost:8081/healthz>
- LLM 网关管理后台：<http://127.0.0.1:18080/admin>（仅本机，Bearer token 鉴权）

首次起栈即见团队工作状态（opc-init 生成脱敏成员骨架 + 欢迎任务）；collector 每 1 分钟增量采集刷新数据。

## 服务组成

| 服务 | 进程 | 端口 | 守护 |
|---|---|---|---|
| llm-gateway | `node server.js` | 18080（仅 127.0.0.1） | systemd `opcos-gateway` |
| opc-api | `node index.cjs` | 3033 | systemd `opcos-api` |
| web | `node scripts/web-serve.js` | 8081 | systemd `opcos-web` |
| collector | `python3 opc-bus.py` + 宿主机 crontab | — | systemd `opcos-collector` + crontab |
| dispatcher | `python3 -m kanban_dispatcher.cli` | — | systemd `opcos-dispatcher` |

- **网关**：统一模型名 `hermes-opc-v1`，熔断 / 月度预算 / 多上游路由在网关内部完成，客户 key 只进 `llm.env`（chmod 600）；
- **collector**：`opc_collect.py`（状态采集）+ `token_stats.py`（token 归因）由宿主机 crontab 每 1 分钟运行，
  另有 **`opc-bus.py` 数据总线守护**（1s 增量轮询 `kanban.db` 的 `task_events`，秒级刷新 `status.json` 的 kanban 段）；
- **dispatcher**：60s tick 弹性调度（按机器容量自动推导并发，`--tier/--mem-mb/--cpus` 可显式覆盖）。

页面只 fetch 本地 `/opc/data/*` JSON，零外连、零第三方端点，数据不出机器。

## 配置（config/config.json）

换机器 / 换团队 **只改配置，不动脚本**：

| 键 | 说明 |
|---|---|
| `collector.profiles` | 成员 profile 列表（数据源扫描范围） |
| `collector.cron_member` | cron job → 成员 / 动作 / 描述 映射（活动流文案） |
| `collector.name_id` / `name_en` / `roles` / `role_en` / `roster` | 成员名册与角色（中文名 / 英文名 / 职能） |
| `collector.paths` | 数据源与输出路径（`hermes_home` / `kanban_db` / `out_status` / `token_stats_out` …；容器视角 `/data` 前缀由 install-services.sh 重写为宿主机绝对路径 → `config.native.json`） |
| `collector.running_since` | 运行天数起点（透明办公室「已运行 N 天」） |
| `bus` | 数据总线守护配置：`poll_seconds`（默认 `1.0`）、`kanban_db`、`status_out`、`bus_state_file`（默认 `<paths.opc_dir>/bus-state.json`）、`skip_kinds`（默认 `["heartbeat"]`） |

## 数据总线

`opc-bus.py`（systemd `opcos-collector` 守护）——零第三方依赖的 Python 守护，让任务看板在两次 1 分钟
全量快照之间保持秒级新鲜：

- 增量轮询 `kanban.db` 的 `task_events`（水位线文件 `<out>/bus-state.json`——重启不重放、不漏事件），
  默认跳过 `heartbeat` 事件；
- 发现状态迁移事件 → 重算 `status.json` 的 kanban 段并打 `sources.kanban` 段级元数据
  （`ts` / `watermark` / `events`），其余段原样保留；
- 原子写（`tmp` + `os.replace`），前端 `fs.watch` / SSE 流永远不会读到半截文件。与
  `opc_collect.py` 的段级写协调 = 「最后写者胜」：bus 只写 kanban 段 + `sources.kanban`，
  collect 写其他段且从不覆盖更新的 bus 段；
- bus 挂 → systemd `Restart=on-failure` 自动拉起；`opc_collect.py` 1 分钟全量快照照常兜底（不劣化）。

**打包边界**：bus 只需 `kanban.db` 可读 + `status.json` 可写——SSE 推送端点
（`/api/opc/stream`）是目标部署的既有基础设施，本包不含。

## 监控 / 告警 / 自愈（v0.5.1）

生产实战（2026-08-27 事故：代理节点抖动 → 投递超时 → 误报「链路断」）验证过的运维组件合入部署包。
**路径参数化**：所有路径支持环境变量覆盖（`OPC_*` / `WD_*` / `SCRIPT_DIR` …），默认值仅是 OPC 生产 home 的兜底——
客户部署请用 env 指向自己的目录（详见 installation-guide §11.5）。

| 组件 | 脚本 | 职责 |
|---|---|---|
| 告警总线 | `scripts/opc-alert-bus.py` | 统一告警线 `alerts/bus.jsonl`（O_APPEND 原子写、定稿 9 字段 schema）；CLI `append/read_new/mark_routed/mark_resolved/count_new/list`；P2 直接 resolved 留痕；fingerprint `source:type:YYYY-MM-DD` 幂等去重 |
| 告警路由 monitor | `scripts/alerts_bus_monitor.py` | 5min cron monitor：稳定 hash 输出，仅变化时唤醒路由 agent → 分级建 kanban 卡；`[STALE>2h]` 自监控；18:50 日报汇总窗口 |
| 投递看门狗 | `scripts/telegram_delivery_watchdog.py` | 24h/15min 滑动窗口错误统计 + `blocked_config` 扫描；no_agent cron 契约：健康静默、异常/恢复才输出 |
| 代理自愈看门狗 | `scripts/singbox-watchdog.sh` | 阶梯自愈（SIGHUP → 重启 → 30min 熔断）；v2.1 持续窗口防瞬态误杀（≥5min 才动 gateway）+ 运行中任务保护 |
| 代理守护 | `systemd/sing-box.service.in` | systemd **user** unit 模板（`%h` 相对路径天然可移植）；`Restart=always` + `StartLimitIntervalSec=0`；安装：`cp … ~/.config/systemd/user/ && systemctl --user enable --now sing-box` |
| 总线自测 | `scripts/test_alert_bus.py` | 24 断言（append 原子性 / P2 / read_new / mark 迁移 / fingerprint / 证据截断 / CLI 冒烟），临时目录隔离零污染 |

系统级监控（system-monitor 降载感知/自愈）随 P0 验收通过后合入（delivery-checklist §2 验收点 21）。

改完重启生效：`scripts/install-services.sh restart --out <opc-init 输出目录>`。

## 数据自持

- 全部数据在 `<opc-init 输出目录>`（`<out>`）里：`kanban.db` / `status.json` / `token-stats.json` /
  `bus-state.json` / `llm-gateway.json` / `hermes-home/` / `llm.env`(600)。
- 备份 / 导出（`scripts/backup.sh`，默认保留最近 14 份到 `backups/<日期>/`）：

```bash
scripts/backup.sh --data-dir <opc-init 输出目录>          # 原生形态推荐
scripts/backup.sh --data-dir <输出目录> --keep 30 --out /mnt/ops   # 自定义保留份数 / 目标目录
```

- 备份内容：`kanban.db`（任务与事件留痕）、`status.json`、`token-stats.json`、`state.json`；
  缺任一文件即失败中止（防不完整备份）。恢复步骤见 installation-guide v0.5 §12。

### 从演示状态切换到真实数据

1. 停栈：`scripts/install-services.sh stop --out <out>`
2. 把真实 `kanban.db` / `state.json` / Hermes profile 数据放入 `<out>`（或改 `config.native.json` 的 `paths` 指向它们）
3. 按你的团队改 `config/config.json` 的 `profiles` / `name_id` / `roster` / `cron_member`，重跑
   `scripts/install-services.sh install --out <out>`（幂等，重写 config.native.json）
4. 起栈验证：`scripts/install-services.sh start --out <out>` + `scripts/smoke_test.sh <端口>`

完整分步操作见 [安装文档 docs/installation-guide.md](docs/installation-guide.md) 与
[部署手册 deploy-guide.md](deploy-guide.md)。

## 版本与升级

- **版本单一来源 = 仓根 `VERSION` 文件**（当前 `0.6.0`）；发布 = VERSION + CHANGELOG 条目 + git tag 三处对齐（semver）。
- 治理层运行在 **Hermes Agent 的锁定 fork 底座**（`theBigGavin/hermes-agent-fork`，私有，锁 tag
  `v0.20.5-opc.1`）之上；底座**每周 merge upstream**，部署包按 git tag 发布。
- **检查更新**：`python3 scripts/opc-os-check-update.py --json`——读本地 VERSION → 探测最新版，离线静默退出；
  有新版时写 `~/.opc-os/update-available.json` + 推送 alert-bus `upgrade-available` 通知（透明办公室角标数据源）；
  由 install-services.sh 的 crontab 块每日定时执行（`OPC_UPDATE_CHECK_INTERVAL` 可配）。
- **一步升级**：`bash scripts/opc-os-upgrade.sh [--to vX.Y.Z]`——备份先行 → 增量升级 → 冒烟 → 失败自动回滚；
  `--dry-run` 预览、`--rollback` 回到上一 tag 并恢复数据。
- **手工/离线兜底**（installation-guide v0.6 §9.5）：备份先行 → `git pull`（或 checkout 新 tag / rsync 部署包）→
  `scripts/install-services.sh install --out <out>` 重跑 → smoke / doctor 验证。
- **全量升级（主版本变化）**：脚本拒绝自动升级，输出全量重装指引（数据保留）；确需强制用 `--force`。
- 回退：`bash scripts/opc-os-upgrade.sh --rollback`，或手工 `git checkout <旧 tag>` 重跑 install。升级与回退都不动数据目录。

## 冒烟测试

```bash
scripts/smoke_test.sh 8081
```

37 项检查（配 `WORKBENCH_API` 时 38 项）：5 服务进程在线、页面/接口全 200、clean-URL 语义、
数据真实流动（成员 / 任务 / token 非写死）、数据总线（opc-bus 存活、`sources.kanban` 元数据、
水位线文件）、脱敏红线（无任何真实身份信息）。

## 自助诊断（doctor）

```bash
cd opc-os && python3 scripts/opc-os-doctor.py          # 10 组检查：依赖/进程/unit/端口/网关/代理/key/派发/数据流/红线
python3 scripts/opc-os-doctor.py --json                # 机器可读 {group,status,detail,fix}
echo $?                                                # 0=全绿 / 1=有 FAIL / 2=工具错误
```

> doctor（`scripts/opc-os-doctor.py`）随 **P0-3 交付**（2026-08-28 排期）合入部署包，合入前以
> installation-guide v0.5 §10 描述的定稿规格为准。

## 常见问题

**运行时需要联网吗？**
不需要。仪表盘只读本地目录里的 JSON。（仅安装期需要包源：node / python / uv，受限网络见 installation-guide §2。）

**数据在哪？**
在 `<opc-init 输出目录>` 目录里（`kanban.db` / `status.json` / `token-stats.json` 等）。
不发送到任何地方。

**想重置回纯净演示状态？**
重新跑一遍 opc-init（`--out` 换新目录或先清空旧目录）再 install-services install。数据目录删除即重置。

**8081 端口被占了？**
`scripts/install-services.sh install --out <out> --web-port <新端口>` 换端口，冒烟测试把新端口
传给 `smoke_test.sh` 即可。

**演示数据是真的吗？**
不是。演示团队、任务、token 数字全部为虚构，且已脱敏。

**服务怎么管理？**
`scripts/install-services.sh status|restart|stop --out <out>`（systemd 环境等价
`systemctl status opcos-*`）；日志 `journalctl -u opcos-<服务>` 与 `<out>/logs/*.log`。

## 数据边界（红线）

- 自带演示数据**全部为虚构**，不含任何真实业务数据。
- 部署实例数据全部内部自持，无外部行情 / 第三方数据混入。
- 本包**不含任何收款 / 计费 / 定价代码**。
- 上游 LLM key 只进 `<out>/llm.env`（chmod 600），文档 / 代码 / 日志零明文凭据。