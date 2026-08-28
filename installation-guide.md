# OPC-OS 安装交付文档（从零到可用）

> 适用版本：v0.6.0（客户形态 · 非 docker 原生进程版，2026-08-28 更新）
> 本文档 = 在一台**全新 Ubuntu 机器**上，把 OPC-OS 客户形态从零装到「团队 Agent 可用」的完整操作手册。
> v0.4 = Gavin 拍板去 docker 化（run 1350 续作）：5 个服务全部改为**原生进程**（systemd 守护，无 systemd 时退化 nohup），
> 依赖只有 node + python3，不再需要 docker / compose / 镜像构建。
> 所有命令均已在 sandbox1（Ubuntu 24.04.4，4 核 / 7.5G / 367G，**LXC 容器**）从零实测通过；标注了版本与耗时。
>
> **v0.5 变更（邵阳架构盘点 GAP-8 补全，2026-08-27）**：
> - 新增 **§9 增量升级 SOP**（备份先行 → 更新部署包 → install-services 重跑 → doctor/smoke 验证）；
> - 新增 **§10 故障诊断**（opc-os-doctor 用法 + 常见故障定位表；doctor 工具见 P0-3 交付，合入前以本文描述为准）；
> - 新增 **§11 日常监控**（systemd 状态 / journalctl 日志 / 关键状态文件位置索引）；
> - 新增 **§12 备份与恢复**（backup.sh 备份集内容 + 恢复步骤）；
> - smoke_test 检查项数随部署包演进同步为 **37 项**（v0.4 写的 32 项为当时版本，见 §7.3）；
> - 全文命令 / 路径 / 端口 / 服务清单与部署包（scripts/、systemd/、config/）逐条核对一致。
>
> **v0.6 变更（Gavin 0827 升级机制指令，2026-08-28）**：
> - **§9 重写为「升级机制」**：`opc-os-check-update.py`（版本检查/通知）+ `opc-os-upgrade.sh`
>   （安全升级：备份先行 → 增量 → 冒烟 → 失败自动回滚）；手工 SOP 降级为兜底（§9.5）；
>   §9.7 回退升级为脚本 `--rollback` + 手工双路径；
> - **§12 备份集扩展 4 → 7 文件**（+gateway.config.json / bus-state.json / govern-state.json，
>   核心缺一中止 / 配置缺一警告）；
> - **版本制度**：仓根 `VERSION` 文件为单一来源，`opc_init.py --version` / check-update / upgrade
>   全部读取（VERSION + CHANGELOG + git tag 三处对齐）。

---

## 0. 交付形态一句话

OPC-OS 客户形态 = **5 个原生进程**（llm-gateway 网关 + opc-api + web 透明办公室 + collector 采集/数据总线 + dispatcher 弹性调度）
+ 宿主机上的 Hermes 团队 Agent 运行时（`install-hermes-worker.sh` 安装）。团队成员的 LLM 调用全部经 llm-gateway 网关
（统一模型名 `hermes-opc-v1`，熔断/预算/多上游路由在网关内部完成），客户 key 只进 `llm.env`（chmod 600），不落任何代码/配置/文档。

| 服务 | 进程 | 端口 | 守护 |
|---|---|---|---|
| llm-gateway | `node server.js`（llm-gateway/） | 18080（仅 127.0.0.1） | systemd `opcos-gateway` |
| opc-api | `node index.cjs`（opc-api/） | 3033 | systemd `opcos-api` |
| web | `node scripts/web-serve.js`（零依赖静态服务器） | 8081 | systemd `opcos-web` |
| collector | `python3 opc-bus.py` + 宿主机 crontab（opc_collect/token_stats/llm_gateway_sync） | — | systemd `opcos-collector` + crontab |
| dispatcher | `python3 -m kanban_dispatcher.cli`（fork venv） | — | systemd `opcos-dispatcher` |

无 systemd 的环境（部分 LXC）自动退化为 `nohup + pidfile` 守护（`install-services.sh start/stop/status`），**LXC 可用是去 docker 化的核心收益**（v0.3 的 docker 版在 LXC 里跑不起来，见 §13）。

---

## 1. 前置要求

| 项 | 要求 | 检查命令 |
|---|---|---|
| 操作系统 | Ubuntu 22.04 / 24.04（实测 24.04.4 LTS；**LXC/真机/KVM 均可**） | `cat /etc/os-release` |
| CPU / 内存 | ≥ 2 核 / ≥ 4G（4 核 7.5G 实测流畅；dispatcher 弹性自动推导并发） | `nproc && free -h` |
| 磁盘 | ≥ 5G 可用（无镜像，依赖极少） | `df -h /` |
| 预装软件 | git + python3.10+（实测 3.12.3）+ node ≥ 20（实测 22.23.2）+ uv（见 §3） | 见下方 |
| 网络 | 能访问外部包源（pypi/npm/astral）；受限网络见 §2 | 见 §2 |

预装检查命令：

```bash
for b in git python3 node uv; do printf "%s: " "$b"; which $b 2>/dev/null || echo "未装"; done
python3 --version        # 需 3.10+（实测 3.12.3）
node --version           # 需 ≥ 20（实测 v22.23.2）
```

> ⚠️ **Ubuntu 24.04 缺 python3-venv**：`python3 -m venv` 默认不可用（ensurepip 缺失），
> install-hermes-worker.sh 会失败。必须先装：`sudo apt install -y python3.12-venv`（实测坑，见 §13 坑 2）。

---

## 2. 外部访问配置（沙箱 / 内网部署环境）★

> 本节适用所有走内网桥的沙箱/隔离部署环境。**统一代理 = http://10.77.0.1:7890（已实测可用）**，
> 搜索后端 = SearXNG http://10.77.0.1:11234（已实测可用）。

### 2.1 翻墙代理（宿主机 mihomo 经内网桥转发）

```bash
# 单次使用
curl -x http://10.77.0.1:7890 https://github.com

# 持久（写入 shell 会话 / ~/.bashrc）
export http_proxy=http://10.77.0.1:7890
export https_proxy=http://10.77.0.1:7890
# sudo 场景需要透传：
sudo -E apt-get install -y <pkg>
```

实测连通性（2026-08-26）：github 200 / google 302 / pypi.org 200 / registry.npmjs.org 200 / astral.sh 200。

> ⚠️ **坑：勿以 10.0.2.2:7890 测试代理连通性**——那是宿主侧地址，沙箱网络后不通（实测 000）。
> 不要因为 10.0.2.2 测不通就得出「无需代理」的错误结论；正确地址是 **10.77.0.1:7890**。

### 2.2 搜索后端（SearXNG）

```bash
网页:     http://10.77.0.1:11234/
JSON API: http://10.77.0.1:11234/search?q=<query>&format=json
```

实测：返回 200 + 正常 JSON 结果。

### 2.3 依赖源可达性（无 docker，只需 pypi/npm）

```bash
curl -s -o /dev/null -w '%{http_code}\n' https://pypi.org/simple/           # 200
curl -s -o /dev/null -w '%{http_code}\n' https://registry.npmjs.org/        # 200
curl -s -o /dev/null -w '%{http_code}\n' https://astral.sh/uv/install.sh    # 200
```

---

## 3. 环境自举（node / uv / git）

> 全部在宿主机（sandbox）上执行。实测耗时括号内。

### 3.1 Node / npm（实测：nodesource LTS，约 85s）

```bash
export http_proxy=http://10.77.0.1:7890 https_proxy=http://10.77.0.1:7890
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt-get install -y nodejs
node --version && npm --version
```

实测版本：node v22.23.2（LTS）+ npm 10.9.8。
> ⚠️ 坑：直接 `apt install nodejs` 装的是 Ubuntu 默认 v18.19.1 且**不带 npm**（半套）；
> 必须走 nodesource 装 22 LTS。装完确认 `node -v` 以 v2x 开头、`npm -v` 有输出。

### 3.2 uv（实测：官方脚本，约 8s）

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
export PATH="$HOME/.local/bin:$PATH"
uv --version
```

实测版本：uv 0.12.5。

### 3.3 python3-venv（Ubuntu 24.04 必装）

```bash
sudo apt-get install -y python3.12-venv
```

### 3.4 网络核验（一条龙）

```bash
for u in https://github.com https://www.google.com https://pypi.org https://registry.npmjs.org https://astral.sh; do
  curl -x http://10.77.0.1:7890 -s -o /dev/null -w "$u -> %{http_code}\n" --connect-timeout 10 "$u"
done
```

通过标准：全部 200（google 302 正常）。

---

## 4. 获取部署包

### 方式 A：rsync 本机资源（推荐，沙箱场景）

```bash
# opc-os 部署仓（含 deploy-guide.md、scripts/、systemd/、vendor/、web/）
rsync -a --exclude opc-init-out --exclude backups --exclude .git <本机>/opc-os/ sandbox1:~/opc-os/
# hermes-agent fork 源码（rsync 排除 .git/venv，install 脚本会自动重建）
rsync -a --exclude .git --exclude venv <本机>/hermes-agent/ sandbox1:~/hermes-agent-src/
```

### 方式 B：git clone（有 GitHub 凭据时）

```bash
git clone <opc-os 私有仓地址> opc-os && cd opc-os
git clone https://github.com/theBigGavin/hermes-agent-fork hermes-agent-src && cd hermes-agent-src && git checkout v0.20.5-opc.1
```

两种方式二选一。fork 源码用于 install-hermes-worker.sh 的 `--source` 参数（绕过 GitHub 凭据，见 §6）。

---

## 5. opc-init 生成客户团队（参数化，非交互）

```bash
cd ~/opc-os
python3 scripts/opc_init.py --template software --name "<客户公司名>" --gateway --out ~/opc-test
```

参数说明：

| 参数 | 含义 |
|---|---|
| `--template` | 行业模板：software / manufacturing / retail / aigc（`--list-templates` 查看） |
| `--name` | 公司/门店名称（必填，会写入团队名册，请用脱敏测试名） |
| `--gateway` | 网关模式（等价 `--llm gateway`）：base_url 指向 llm-gateway，熔断/月度预算由网关负责 |
| `--gateway-url` | 网关地址（默认 http://127.0.0.1:18080/v1，无需改） |
| `--out` | 输出目录（默认 ./opc-init-out） |
| `--force` | 已有产物时覆盖（覆盖前自动备份） |

产物清单（实测，`~/opc-test/`）：

```
config/config.json        配置（collector 零改动直接读；paths 为容器视角 /data，由 install-services.sh 映射为 config.native.json）
hermes-home/profiles/<id>/  成员骨架（SOUL.md / persona.md / role.md / skills.md / cron/ / config.yaml）
state.json                团队名册 + 决策分级（decided_by）
kanban.db                 空板 + 欢迎任务（CEO）+ 初扫任务（情报）
init-state.json           初始化标记（=initialized，首启检测用）
llm.env                   LLM key + 网关 client token（chmod 600）
init-report.txt/json      初始化报告
```

software 模板 9 角色：CEO、开发、运维、营销、客服、财务、情报、测试、设计。脱敏自检通过（产物零 OPC 真实身份）。

---

## 6. install-hermes-worker（客户 Hermes 运行时）

### 6.1 配置 llm.env（上游 key，只进 llm.env）

```bash
# 把上游 key 合并进 opc-init 生成的 ~/opc-test/llm.env（保留 GATEWAY_CLIENT_TOKEN 不覆盖）
# 从本机 gateway.env / gateway.config.json 提取 providers 的 api_key_env 对应变量：
#   deepseek → DEEPSEEK_API_KEY、kimi → KIMI_API_KEY、openrouter → OPENROUTER_API_KEY
# 网关 admin 用 GATEWAY_ADMIN_TOKEN（compose 引用 GATEWAY_ADMIN_TOKEN=${GATEWAY_ADMIN_TOKEN:-}）
# 示例（在 sandbox 上执行；key 绝不落聊天/代码/文档）：
#   scp <本机>:gateway.env sandbox1:~/gw_env_tmp
#   for name in DEEPSEEK_API_KEY KIMI_API_KEY OPENROUTER_API_KEY GATEWAY_ADMIN_TOKEN; do
#     val=$(grep "^${name}=" ~/gw_env_tmp | cut -d= -f2-); echo "${name}=${val}" >> ~/opc-test/llm.env
#   done
#   chmod 600 ~/opc-test/llm.env && rm -f ~/gw_env_tmp   # 临时副本必删
```

llm.env 最终含：`OPC_LLM_PROVIDER / OPC_LLM_MODEL / OPC_LLM_BASE_URL / OPC_LLM_API_KEY（可空）/ GATEWAY_CLIENT_TOKEN（opc-init 随机生成）/ DEEPSEEK_API_KEY / KIMI_API_KEY / OPENROUTER_API_KEY / GATEWAY_ADMIN_TOKEN`，权限 600。

### 6.2 安装脚本

```bash
export http_proxy=http://10.77.0.1:7890 https_proxy=http://10.77.0.1:7890   # 依赖下载走代理
# 方式一：本地源码包（推荐，无 GitHub 凭据）
bash ~/opc-os/scripts/install-hermes-worker.sh \
  --source ~/hermes-agent-src \
  --hermes-home ~/opc-test/hermes-home \
  --gateway-url http://127.0.0.1:18080/v1
# 方式二：私有仓拉取（有 GITHUB_TOKEN 时）
GITHUB_TOKEN=<pat> bash ~/opc-os/scripts/install-hermes-worker.sh \
  --hermes-home ~/opc-test/hermes-home \
  --gateway-url http://127.0.0.1:18080/v1
```

脚本行为（红线：一律官方命令，不手写 config.yaml）：
1. 装 hermes-agent fork（锁 tag v0.20.5-opc.1；`--source` 模式拷贝源码 + 本地 git init 打快照 tag，幂等重跑跳过安装）；
2. 为每个 profile 用 `hermes --profile <id> config set` 写：model.default=`hermes-opc-v1`、model.provider=`custom:hermes-opc`、model.base_url=网关、model.api_key=`${GATEWAY_CLIENT_TOKEN}`（引用，无明文）、custom_providers 块、reasoning_echo=true、fallback_providers=[]、timezone；
3. 上游 key 只进 llm.env（chmod 600），网关 GATEWAY_ENV_FILE 引用；
4. 完成校验：hermes --version + profiles 列表 + 网关 /v1/models。

参数说明：`--tag` 换 tag=升级；`--force` 强制重装；`--install-dir` 默认 ~/.opc-os/hermes-agent；`--llm-env` 默认 hermes-home 父目录/llm.env。

> ⚠️ 坑：**必须先装 python3.12-venv**（§3.3），否则 venv 创建失败 rc=1（实测踩到）。
> ⚠️ 首装失败要检查脚本退出码与日志（`tail ~/install-worker.log`），勿把「校验段静默跳过」当成功。

### 6.3 验证安装

```bash
export HERMES_HOME=~/opc-test/hermes-home
~/.opc-os/hermes-agent/venv/bin/hermes --version          # 有版本输出
cd ~/.opc-os/hermes-agent && git describe --tags --exact-match   # v0.20.5-opc.1
ls ~/opc-test/hermes-home/profiles/                        # 9 个成员目录
```

---

## 7. 起栈与配置（install-services.sh，替代 docker compose）

### 7.1 一键安装（幂等，可重跑）

```bash
cd ~/opc-os
scripts/install-services.sh install --out ~/opc-test
```

实测耗时：**3.3s（重跑幂等）**。脚本自动完成：

1. **路径映射**：opc-init 的 config.json 是容器视角（paths 全 `/data/...`），脚本生成 `config/config.native.json`（/data → 宿主机绝对路径，json 级安全替换，实测重写 9 项）；
2. **目录就绪**：`<out>/logs`、`<out>/run`、`<out>/llm-gateway`（网关事件落盘）、`<out>/workspaces`、`<out>/attachments`；
3. **网关数据目录**：首次拷贝默认 `gateway.config.json`（管理面板热改落此文件）；
4. **secrets.env**（chmod 600）：`WORKBENCH_TOKEN`（随机生成，工作台写回用）+ `GATEWAY_CLIENT_TOKEN`（从 llm.env 提取，dispatcher worker 鉴权用）——幂等，重跑只补缺；
5. **env.sh**（chmod 600）：成员交互环境（HERMES_HOME / KANBAN env / GATEWAY_CLIENT_TOKEN 导出），source 后直接 `hermes -p <成员id>`；
6. **crontab**：`*/1` opc_collect.py + token_stats.py、`*/5` llm_gateway_sync.py（幂等，先删旧块再写）；
7. **启动**：检测到 systemd → 渲染 `systemd/*.service.in` → `/etc/systemd/system/opcos-*.service` + enable + **restart**（enable --now 对已 active 服务幂等跳过，故显式 restart）；无 systemd → nohup+pidfile 守护。

可选参数：

| 参数 | 默认 | 说明 |
|---|---|---|
| `--web-port / --api-port / --gateway-port` | 8081 / 3033 / 18080 | 端口覆盖 |
| `--tier small\|medium\|large` | auto | 弹性档位 → KANBAN_DISPATCHER_TIER |
| `--mem-mb N / --cpus N` | 空 | 容量精确覆盖（KANBAN_DISPATCHER_CAPACITY_*） |
| `--workbench-api URL` | 不注入 | 注入 workbench.html 的 API base |
| `--hermes-agent-src <目录>` | ~/.opc-os/hermes-agent | fork 安装目录 |
| `--systemd / --no-systemd` | auto | 强制/禁用 systemd 守护 |

### 7.2 服务管理

```bash
scripts/install-services.sh status --out ~/opc-test     # systemd 状态 / pidfile 状态 + crontab
scripts/install-services.sh restart --out ~/opc-test    # 全部重启（改配置后用它，重渲染 unit 用 install）
scripts/install-services.sh stop --out ~/opc-test
scripts/install-services.sh uninstall --out ~/opc-test   # 停服务 + 移除 units/crontab（数据保留）
```

### 7.3 起栈后一键冒烟

```bash
cd ~/opc-os && scripts/smoke_test.sh 8081
```

通过标准：**37/37 ALL GREEN**（进程/HTTP/clean-url/workbench/数据/总线/脱敏红线七组；配了 WORKBENCH_API 时为 38/38）。

### 7.4 LLM 网关管理后台（运维入口，可选）

> llm-gateway 内置完整管理面板（0825-i/j 起）：浏览器静态壳 + admin API，含 provider 池管理 / 熔断阈值 / 月度预算 / 余额监控 / 配置热改。
> 所有变更原子写 `gateway.config.json`（自动 `.bak` 备份）+ **热生效无需重启**，操作留痕 `events.jsonl`（operator 只记 env 变量名，永不记 token 值）。

**访问方式**（仅本机）：

```bash
浏览器:  http://127.0.0.1:18080/admin
命令行:  curl -H "Authorization: Bearer <GATEWAY_ADMIN_TOKEN>" http://127.0.0.1:18080/admin/status
```

**鉴权**：所有 `/admin/*` API 必须带 `Authorization: Bearer <GATEWAY_ADMIN_TOKEN>`（无 token → 401；静态壳页面 `/admin` 本身可打开，数据与操作全在需鉴权的 API）。token 来源 = `<out>/llm.env` 中的 `GATEWAY_ADMIN_TOKEN`（§6.1 已说明如何把上游 key 与 admin token 合并进 llm.env；网关进程经 `GATEWAY_ENV_FILE=<out>/llm.env` 注入，也可直接放环境变量）。查看是否已配置（**勿贴值**）：

```bash
grep -q '^GATEWAY_ADMIN_TOKEN=.' ~/opc-test/llm.env && echo 'admin token 已配置' || echo '未配置（admin API 全部 401）'
```

**安全边界（红线）**：网关默认只绑 `127.0.0.1:18080`（gateway.config.json host/port 即默认值），管理面板含配置与密钥管理入口，**禁止裸暴露公网**。远程管理建议 SSH 隧道：

```bash
ssh -L 18080:127.0.0.1:18080 <user>@<host>    # 本机浏览器开 http://127.0.0.1:18080/admin
```

或内网反代 + token 鉴权；改监听地址须显式设 `GATEWAY_HOST` env（如容器内 0.0.0.0），并自行保证网络隔离。

**管理 API 一览**（全部需 Bearer token）：

| 方法 & 路径 | 作用 |
|---|---|
| `GET /admin/status` | 池总览：provider 健康/熔断/余额/用量、熔断参数、切换历史、config 内存↔磁盘同步状态 |
| `GET /admin/keys` | BYOK key 清单（仅状态：env 变量名/是否已配置，**永不回 key 值**） |
| `PUT /admin/provider/default` | `{"name": "..."}` 设置默认 provider |
| `PUT /admin/thresholds` | 熔断/健康探测/月度预算/余额阈值 |
| `POST /admin/providers` | 新增 provider |
| `POST /admin/providers/fetch-models` | 一键拉取上游 /models 列表 |
| `POST /admin/providers/test` | 直连上游试聊 |
| `PUT / DELETE /admin/providers/:name` | 更新 / 移除 provider（池中须保留至少一个） |

**验证（照抄可通，2026-08-26 本机实测）**：

```bash
curl -s http://127.0.0.1:18080/healthz                      # → ok（无需 token）
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:18080/admin/status   # 无 token → 401
ADMIN_TOKEN=$(grep '^GATEWAY_ADMIN_TOKEN=' ~/opc-test/llm.env | cut -d= -f2-)
curl -s -o /dev/null -w '%{http_code}\n' -H "Authorization: Bearer $ADMIN_TOKEN" http://127.0.0.1:18080/admin/status   # 带 token → 200
```

通过标准：依次 `ok` / `401` / `200`。浏览器打开 http://127.0.0.1:18080/admin 可看到面板（首次需输入同一 token）。

---

## 8. 验证清单（部署后逐项过，全部已实测）

### 8.1 网关

```bash
TOKEN=$(grep '^GATEWAY_CLIENT_TOKEN=' ~/opc-test/llm.env | cut -d= -f2-)
curl --oauth2-bearer "$TOKEN" http://127.0.0.1:18080/v1/models
```

通过标准：200 且响应仅 `hermes-opc-v1` 一个模型（无上游透传）。实测：HTTP 200，0.92s。
真实调用：`curl --oauth2-bearer "$TOKEN" -H "Content-Type: application/json" http://127.0.0.1:18080/v1/chat/completions -d '{"model":"hermes-opc-v1","messages":[{"role":"user","content":"只回复两个字：你好"}],"max_tokens":64}'` → 200 + 正常回复（走上游 key，实测路由 deepseek 系）。

### 8.2 成员走网关

```bash
source ~/opc-test/env.sh        # 关键！导出 HERMES_HOME + GATEWAY_CLIENT_TOKEN（缺了必 401，见 §13 坑 4）
~/.opc-os/hermes-agent/venv/bin/hermes -p <成员id> chat -q "只回复一句话：链路 OK"
```

通过标准：正常启动，经网关对话成功。实测：4s 返回，会话落盘 `<hermes-home>/profiles/dev/state.db`（token 归因数据源）。

### 8.3 kanban 跑通（建卡 → 派活 → 完成）

```bash
source ~/opc-test/env.sh
~/.opc-os/hermes-agent/venv/bin/hermes kanban create "验证卡" --assignee dev --body "直接回复一句话确认后 kanban_complete"
# dispatcher 60s tick 自动派活；观察：
~/.opc-os/hermes-agent/venv/bin/hermes kanban list
sudo journalctl -u opcos-dispatcher --no-pager -n 10 | grep tick    # spawned=N 增长
```

通过标准：任务 ready → running（worker 被 spawn，命令行 `hermes -p <成员> --cli chat -q work kanban task <id>`）→ done。
实测：建卡 11:42:54 → spawn 11:45:44（含 60s tick 间隔）→ 完成 11:45:52（worker 8s 内走网关回复 + complete）。

### 8.4 cron / 总线

```bash
crontab -l | grep -E 'opc_collect|llm_gateway_sync'   # */1 采集 + */5 网关同步两条（命令本体；"OPC-OS" 只出现在块标记行）
pgrep -af opc-bus.py                        # 有 pid（数据总线守护存活）
# 手动触发一次（不等周期）：
cd ~/opc-os/scripts && OPC_OS_CONFIG=~/opc-test/config/config.native.json python3 opc_collect.py
ls -la ~/opc-test/status.json ~/opc-test/token-stats.json ~/opc-test/bus-state.json
```

实测：crontab 两条就位；opc-bus 存活；手动采集成功（tokens 18807 / sessions 3 / 9 成员归因）。

### 8.5 业务链路

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8081/opc/           # 透明办公室 200
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8081/opc/governance.html   # 治理仪表盘 200
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8081/opc/workbench.html    # 工作台 200
curl -s http://127.0.0.1:8081/opc/data/status.json        # 含 sources.kanban（ts/watermark/events）与 kanban.summary
curl -s http://127.0.0.1:8081/opc/data/token-stats.json   # 含 tokens/cost_usd/pass_rate/tasks_done
cd ~/opc-os && scripts/smoke_test.sh 8081                 # 37/37 全绿
```

### 8.6 运维基建

```bash
ls -la ~/opc-os/scripts/backup.sh                     # 可执行
bash ~/opc-os/scripts/backup.sh --data-dir ~/opc-test # 实测：备份 kanban.db/status.json/token-stats.json/state.json → backups/<日期>/
sudo journalctl -u opcos-dispatcher --no-pager -n 30 | grep capacity   # 单 tick 日志含 capacity derived
```

实测 capacity 推导：`capacity derived: meminfo=8000MB cgroup=NoneMB cpu_quota=None cores=4 cgroup_detected=False -> effective_mem=8000MB mem_workers=8 cpu_workers=4 max_in_progress=4 per_profile=2 (source=derived)`（裸机探测 fallback，LXC 无 cgroup 也正常）。

### 8.7 红线复核

```bash
grep -rE 'sk-[A-Za-z0-9]{32,}' ~/opc-test/hermes-home/profiles/ --exclude-dir=bin     # 无输出（上游 key 只在 llm.env）
stat -c %a ~/opc-test/llm.env ~/opc-test/secrets.env   # 600 / 600
grep -riE 'Gavin|庄子|马克|林静' ~/opc-test/        # 无输出（零真实团队身份）
```

### 8.8 EnvironmentFile 约束核查（P2 固化）

**为什么需要**：5 个 `opcos-*.service` unit 全部经 systemd `EnvironmentFile` 注入环境变量——裸 node/python 守护进程**不自行加载 env**，`GATEWAY_CLIENT_TOKEN` / `WORKBENCH_TOKEN` 的引用（如 profile `api_key=${GATEWAY_CLIENT_TOKEN}`）依赖这个注入面，缺失即 401（§10.2 / §13 坑 4）。渲染来源优先级：`--env-file` 显式指定 > `<out>/secrets.env`（opc-init 生成）> `~/.hermes/.env`（生产 Hermes 主环境）。

```bash
# ① 5 个 unit 实线必须齐全（每个 1 行）
grep -c '^EnvironmentFile=' /etc/systemd/system/opcos-*.service    # 应 5 个文件各 1

# ② 指向的 env 文件存在且权限 600
grep -h '^EnvironmentFile=' /etc/systemd/system/opcos-*.service | cut -d= -f2- | sort -u | xargs -r stat -c '%a %n'   # 600 <路径>

# ③ 关键 token 在注入面内
EF=$(grep -h '^EnvironmentFile=' /etc/systemd/system/opcos-*.service | head -1 | cut -d= -f2-)
grep -q '^GATEWAY_CLIENT_TOKEN=.' "$EF" && echo 'token ok'

# ④ doctor 模板一致性（组 3：模板 EnvironmentFile 约束 + Restart 策略）
python3 ~/opc-os/scripts/opc-os-doctor.py --strict --group g3
```

> unit 变更后 `sudo systemctl daemon-reload && sudo systemctl restart opcos-*` 生效；`scripts/install-services.sh install --out <out>` 重跑即重新渲染（幂等，自动解析注入面）。

---

---

## 9. 升级机制（check-update + upgrade.sh 产品化升级）

> 适用：部署已跑起来（服务 active、数据在 <out> 里），部署包出了新版本想升级。
> v0.6 起从「手工 SOP」升级为**产品化机制**：`scripts/opc-os-check-update.py` 负责**发现新版本并通知**，
> `scripts/opc-os-upgrade.sh` 负责**安全升级**（备份先行 → 增量 → 冒烟 → 失败自动回滚）。手工 SOP 降级为**兜底**
> （离线 / rsync 部署场景，见 §9.5）。
> 原则：**备份先行 → 更新 → install-services 重跑 → 验证**；升级全程**不动数据目录**
> （<out> 里的 kanban.db / JSON / profiles / llm.env 全部保留）。

### 9.1 版本制度（VERSION 文件 + CHANGELOG + git tag 三处对齐）

- **版本单一来源 = 仓根 `VERSION` 文件**（内容如 `0.6.0`）。`opc_init.py --version`、`opc-os-check-update.py`、
  `opc-os-upgrade.sh` 全部读取 VERSION 文件，不解析 Python 源码。
- **发布流程**：改 `VERSION` → 在 `CHANGELOG.md` 加条目（版本号 `vX.Y.Z`，semver 主.次.补丁）→ 打 git tag `vX.Y.Z`。
  三处必须对齐，否则 check-update 的 semver 比较会失真。
- 版本基线说明见 CHANGELOG 头部（历史 tag v0.1–v0.3.5 为制度落地前旧打标；tag 发布规则由 G 卡
  G-2026-08-27-002 拍板）。

### 9.2 版本检查与通知（opc-os-check-update.py）

职责：读本地 VERSION → 探测远端最新版 → semver 比较 → 有新版时报**双落点通知**；无新版/离线**静默退出 0**。

```bash
# 手动检查一次（--json 输出机器可读）
cd ~/opc-os && python3 scripts/opc-os-check-update.py --json

# 忽略间隔限制强制检查（默认距上次 < interval 直接跳过）
python3 scripts/opc-os-check-update.py --force --json

# 帮助
python3 scripts/opc-os-check-update.py --help
```

**远端版本源优先级**（全失败 = 静默退出 0 + 记录 last-check，离线不打扰，下次再试）：
1. GitHub Releases API `https://api.github.com/repos/theBigGavin/opc-os/releases/latest`（公开仓免凭据，超时 10s）→ `tag_name`；
2. `git ls-remote --tags origin`（git 形态部署 fallback，需仓内 git remote）。

**通知双落点**（有新版时）：
1. `alert-bus`：`scripts/opc-alert-bus.py append` type=`upgrade-available`（severity=P2 信息留痕，含 current/latest/变更摘要），走既有告警路由；
2. `~/.opc-os/update-available.json`：`{current, latest, changelog_summary, detected_at}`——透明办公室 web 读取显示「新版本可升级」角标数据源。

**定时检查**：由 `install-services.sh` 的 crontab 块统一写入（默认每日 04:30），间隔 env
`OPC_UPDATE_CHECK_INTERVAL` 可配（`daily`/`hourly`/`Nh`/`Nd`/原始 cron）；脚本侧 `OPC_UPDATE_CHECK_INTERVAL_H`
（小时，默认 24）控制间隔跳过。检查日志 `<out>/logs/check-update.log`。

**env 参数化**（D7 合入规范，§14 红线 7）：`OPC_OS_REPO` / `OPC_OS_STATE_DIR`（默认 `~/.opc-os`，update-available.json / last-check.json 落此）/ `OPC_ALERT_DIR` / `OPC_UPDATE_CHECK_URL` / `OPC_UPDATE_CHECK_INTERVAL_H`。

### 9.3 安全升级（opc-os-upgrade.sh）

```bash
# 帮助
bash scripts/opc-os-upgrade.sh -h

# 升级到指定版本（默认 = check-update 检测到的最新版）
bash scripts/opc-os-upgrade.sh --to v0.6.0

# 只做前置检查 + 目标解析，不实际升级（幂等预览）
bash scripts/opc-os-upgrade.sh --dry-run

# 手工回滚到上一可用 tag + 恢复升级前备份
bash scripts/opc-os-upgrade.sh --rollback
```

**阶段化流程**（7 阶段，任一步失败按策略处理，成功才继续）：
1. **前置检查**：部署形态判定（git / rsync）、git 状态干净（本地改动告警，`--force` 跳过）、磁盘空间（≥1G 可用）、当前版本；
   - rsync 部署（非 git 目录）**无法自动增量升级/回滚** → 脚本中止，用部署源重新同步（§9.5）。
2. **自动备份**：调 `backup.sh`（7 文件集，`--keep 3 --out $STATE_DIR/backups`），备份失败 → 中止升级（备份先行红线）。
3. **目标版本解析**：默认 = check-update 检测到的最新版（`update-available.json`）；`--to` 显式指定；无则 git tag 探测最新 semver。
4. **增量/全量判定**：主版本号相同 → 增量（git fetch + checkout tag + `install-services.sh install` 重跑 + 依赖检查）；**主版本号不同 → 拒绝自动升级**，输出全量重装指引（§9.4），除非 `--force`。
5. **冒烟验证**：`smoke_test.sh`（存在才跑）+ `systemctl is-active opcos-*` 全部 active；
6. **失败自动回滚**：checkout 前一个可用 tag + install-services 重跑 + 从备份恢复数据文件（对齐 FORK.md 回退预案：回退只影响代码，数据用升级前备份恢复）；
7. **成功留痕**：`~/.opc-os/upgrade.log` 追加一行 `{ts, from, to, result:ok}`；清除 `update-available.json`。

**env 参数化**：`OPC_OS_REPO` / `OPC_OS_STATE_DIR`（upgrade.log / update-available.json 落此）/ `OPC_DATA_DIR`
（备份数据源，传给 backup.sh `--data-dir`；未设则升级中止）/ `OPC_GATEWAY_CONFIG` / `OPC_SMOKE_PORT`（默认 8081）/
`OPC_UPGRADE_CHECK_DRY`（测试注入冒烟失败）/ `OPC_SMOKE_SKIP_SYSTEMCTL`（无 systemd 环境跳过 is-active 检查）。

### 9.4 全量升级（主版本变化）

脚本**不自动执行**，输出指引后退出 1（除非 `--force`）。手动全量路径（数据保留）：

```bash
# ① 备份已自动完成（$STATE_DIR/backups/<日期>/）
# ② 获取新版部署包（rsync 或 git pull 到新目录）
# ③ 重跑 opc-init（--out 指定原数据目录，保留现有数据）
python3 scripts/opc_init.py --template <同款模板> --out ~/opc-test --force
# ④ 重跑 install-services（复用原数据目录）
scripts/install-services.sh install --out ~/opc-test
# ⑤ 冒烟验证
scripts/smoke_test.sh 8081
```

### 9.5 手工升级 SOP（兜底：离线 / rsync 部署）

> v0.6 起产品化机制优先；以下手工路径在**非 git 部署 / 离线环境**下兜底。原则不变：备份先行 → 更新 → install 重跑 → 验证。

```bash
# ① 备份先行（红线：升级前必做，出问题可回滚数据）
cd ~/opc-os && scripts/backup.sh --data-dir ~/opc-test

# ② 更新部署包（三选一）
git pull                                    # 方式 A：跟 main 分支
git fetch --tags && git checkout <新 tag>    # 方式 B：锁定发布 tag（推荐，跟随 tag 升级）
rsync -a --exclude .git --exclude backups <本机>/opc-os/ ~/opc-os/   # 方式 C：沙箱 rsync

# ③ install-services 重跑（幂等：渲染最新 unit → enable + restart；crontab 重写；缺失文件补齐）
scripts/install-services.sh install --out ~/opc-test

# ④ 验证
scripts/smoke_test.sh 8081                   # 37/37 ALL GREEN
sudo systemctl is-active opcos-gateway opcos-api opcos-web opcos-collector opcos-dispatcher   # 全 active
```

install 重跑的关键语义（v0.4 实测）：`start = enable + restart`——即使服务已在跑也会按**最新渲染的 unit** 重启，
不会因 `enable --now` 幂等跳过而残留旧配置（旧坑见 §13 坑 6）。

### 9.6 fork 底座升级（install-hermes-worker.sh --tag）

治理层运行在 hermes-agent fork 上（锁 tag v0.20.5-opc.1）。底座升级 = 换 tag 重跑安装脚本（幂等，同 tag 跳过、换 tag 升级）：

```bash
bash ~/opc-os/scripts/install-hermes-worker.sh \
  --source ~/hermes-agent-src --tag <新 tag> \
  --hermes-home ~/opc-test/hermes-home \
  --gateway-url http://127.0.0.1:18080/v1
cd ~/.opc-os/hermes-agent && git describe --tags --exact-match   # 确认已切到新 tag
scripts/install-services.sh restart --out ~/opc-test             # 让 dispatcher 用上新底座
```

> 注意：换 tag 升级前同样建议先备份（§9.5 ①）；升级后逐个 profile 对话冒烟（§8.2）。

### 9.7 回退（脚本 --rollback + 手工双路径）

**双路径**（v0.6 起脚本优先）：

```bash
# 路径 A：脚本回滚入口（回到上一可用 tag + 从备份目录自动恢复数据文件）
bash scripts/opc-os-upgrade.sh --rollback

# 路径 B：手工回退（离线 / rsync 部署兜底）
git checkout <上一个可用 tag>     # 回到旧版本部署包
scripts/install-services.sh install --out ~/opc-test   # 重跑 install 即回退（数据不动）
```

如需**连数据一起回退**：`--rollback` 会从备份目录自动恢复数据文件（核心 4 + 配置 3）；手动路径则先从
§12 的备份里把对应日期的数据文件复制回 <out>，再重跑 install。

---

## 10. 故障诊断（opc-os-doctor + 常见故障定位表）

> ⚠️ **doctor 交付状态**：`scripts/opc-os-doctor.py` 随 **P0-3 卡交付**（排期 2026-08-28）合入部署包，
> 当前（2026-08-27）尚未合入。本文按 **P0-3 定稿规格**描述其用法与 10 组检查；脚本落地后以 `--help` 为准，
> 未合入期间用 §10.2 手动命令兜底。doctor 输出零明文凭据（key 一律掩码）。

### 10.1 opc-os-doctor 用法（P0-3 定稿规格）

```bash
cd ~/opc-os
python3 scripts/opc-os-doctor.py              # 人类可读报告：每组 PASS/FAIL/WARN + 一行状态 + 修复建议
python3 scripts/opc-os-doctor.py --json       # 机器可读：{group, status, detail, fix} 结构化数组
echo $?                                       # 退出码：0=全绿 / 1=存在 FAIL（--strict 时 WARN 也计入）/ 2=工具自身错误或环境不可判定
```

常用参数（与 P0-3 交付的 `--help` 一致）：`--out` 指定 opc-init 输出目录（默认自动探测）、`--repo` 部署包仓库根、
`--proxy` 代理地址（默认环境变量或 127.0.0.1:7890）、`--threshold-min` 数据新鲜度阈值分钟（默认 5）、
`--strict`（WARN 计入失败）、`--deep`（真实模型探话，消耗极少 token）、`--group <1-10>`（只跑指定组，排查用）。

**10 组检查含义**：

| # | 检查组 | 判定内容 |
|---|---|---|
| 1 | 运行时依赖 | node ≥ 20 / python3.10+ / uv / git 版本与存在性 |
| 2 | 进程存活 | 5 服务进程 pgrep（llm-gateway / opc-api / web-serve / collector / dispatcher） |
| 3 | systemd unit 状态 | opcos-gateway/api/web/collector/dispatcher 5 unit = active + enabled + Restart 策略（模板与实机都查） |
| 4 | 端口监听 | 18080（仅 127.0.0.1）/ 3033 / 8081 监听者正确 |
| 5 | 网关可达 | health / 探针端点 HTTP 200 + 配置校验（default provider 存在、模型名有效） |
| 6 | 代理连通 | 按配置探测代理（默认 127.0.0.1:7890 或环境指定），curl -x 探测外网可达性 |
| 7 | 模型 key 有效 | llm.env 存在 + 非占位符 + 权限 600 + 网关侧探测（key 只显示掩码 sk-***） |
| 8 | kanban 派发链路 | dispatcher 存活 + lock 文件 / 派发机制健康 + 看板 DB 可读写 |
| 9 | collector 数据流 | token_stats / 状态数据文件新鲜度（mtime 距今 < 阈值）+ 数据总线状态 |
| 10 | 红线检查 | 无明文 key 落代码 / 配置 / 文档、凭据文件权限 600、无裸进程（除文档化 nohup 退化路径） |

### 10.2 常见故障定位表（现象 → doctor 组 → 修复命令）

| 现象 | doctor 组 | 修复命令（均已在 sandbox1 实测） |
|---|---|---|
| 服务起不来（页面/API 打不开） | 组 2 进程 / 组 3 unit | `systemctl status opcos-*` 或 `scripts/install-services.sh status --out ~/opc-test`；`journalctl -u opcos-<服务> --no-pager -n 30` 看报错 |
| 端口被占（8081/3033/18080） | 组 4 端口监听 | `ss -tlnp \| grep <port>` 确认占用者；`scripts/install-services.sh install --out ~/opc-test --web-port <新端口>` 换端口重装 |
| 数据不新鲜（页面数字旧） | 组 9 数据流 | `crontab -l \| grep OPC-OS` 确认采集任务在；`tail -20 ~/opc-test/logs/collect.log` 看采集报错；手动补跑 `cd ~/opc-os/scripts && OPC_OS_CONFIG=~/opc-test/config/config.native.json python3 opc_collect.py` |
| 网关 /v1/models 401 | 组 5 网关 / 组 7 key | `curl -s http://127.0.0.1:18080/healthz`（应 ok）；`source ~/opc-test/env.sh && echo $GATEWAY_CLIENT_TOKEN` 确认有值（用 client token 不是上游 key） |
| 成员对话 401（AuthenticationError）★ 实测坑 | 组 7 key | profile 的 `model.api_key=${GATEWAY_CLIENT_TOKEN}` 是 env 引用：`source ~/opc-test/env.sh` 后再跑（§13 坑 4）；dispatcher worker 侧由 **EnvironmentFile** 注入（默认 `<out>/secrets.env`，`--env-file` 可覆盖，§8.8） |
| kanban 卡一直 ready 不派活（`skipped_nonspawnable=N`）★ 实测坑 | 组 8 派发链路 | dispatcher unit 缺 `Environment=HERMES_HOME=<out>/hermes-home`（§13 坑 5）；改完重跑 `install` 生效 |
| 页面 200 但数据空白 | 组 9 数据流 | `ls -la ~/opc-test/status.json` 确认已生成；`pgrep -af opc-bus.py` 确认总线存活；等下一个 1 分钟采集周期 |
| 网关 admin 面板 401 | 组 5 网关 | `grep -q '^GATEWAY_ADMIN_TOKEN=.' ~/opc-test/llm.env && echo ok`；admin token 未合并进 llm.env 时按 §6.1 合并 |
| LXC 精简环境无 systemd | 组 3（WARN 不判 FAIL） | install 自动退化 nohup+pidfile；管理用 `scripts/install-services.sh start/stop/status`（§13 坑 9） |
| 代理不通（curl 000） | 组 6 代理连通 | 沙箱内网正确代理是 `http://10.77.0.1:7890`，勿用宿主侧 `10.0.2.2:7890`（§13 坑 1） |
| 想定位到具体组件日志 | — | 见 §11.2 journalctl 与 §11.3 状态文件位置表 |

---

## 11. 日常监控（systemd / journalctl / 状态文件）

> <out> = opc-init 输出目录（示例 ~/opc-test）。

### 11.1 服务状态

```bash
# 总览（5 服务 active + crontab 摘要）
scripts/install-services.sh status --out ~/opc-test

# systemd 原生视角
systemctl status opcos-gateway opcos-api opcos-web opcos-collector opcos-dispatcher
systemctl is-active opcos-gateway opcos-api opcos-web opcos-collector opcos-dispatcher   # 逐行 active

# 无 systemd 退化环境
scripts/install-services.sh status --out ~/opc-test     # 显示 run/*.pid 与各进程存活
```

### 11.2 日志查看（journalctl）

```bash
journalctl -u opcos-gateway --no-pager -n 30        # 网关（注意：日志零明文 key）
journalctl -u opcos-api --no-pager -n 30            # 透明办公室/工作台 API
journalctl -u opcos-web --no-pager -n 30            # 静态服务
journalctl -u opcos-collector --no-pager -n 50      # 数据总线（启动行含 poll=1.0s；刷新行含 kanban refreshed）
journalctl -u opcos-dispatcher --no-pager -n 30     # 调度（capacity derived / spawned=N / skipped_nonspawnable）
journalctl -u opcos-dispatcher -f                   # 实时跟随
```

crontab 采集日志（不经 journald，直接落文件）：

```bash
tail -f ~/opc-test/logs/collect.log          # */1 状态采集（opc_collect.py + token_stats.py）
tail -f ~/opc-test/logs/tokenstats.log       # */1 token 归因
tail -f ~/opc-test/logs/llm-gateway-sync.log # */5 网关同步
```

无 systemd 退化环境的服务日志在 `<out>/logs/<服务名>.log`（nohup 重定向）。

### 11.3 关键状态文件位置索引

| 文件 | 内容 | 刷新方式 |
|---|---|---|
| `<out>/status.json` | 状态快照（透明办公室数据源，含 kanban 段 + sources.kanban 元数据） | */1 采集；kanban 段由 opc-bus 秒级刷新 |
| `<out>/token-stats.json` | token 归因（tokens/cost_usd/pass_rate/tasks_done） | */1 |
| `<out>/bus-state.json` | 数据总线水位线（重启不重放、不漏事件） | opc-bus 实时 |
| `<out>/kanban.db` | 任务/事件留痕（透明办公室 + dispatcher 共读） | 实时 |
| `<out>/llm-gateway.json` | 网关同步快照（llm_gateway_sync.py 产物） | */5 |
| `<out>/llm-gateway/gateway.config.json` | 网关运行时配置（管理面板热改落此文件，自动 .bak） | 变更时 |
| `<out>/llm-gateway/events.jsonl` | 网关事件留痕（只记 env 变量名，永不记 token 值） | 实时 |
| `<out>/llm.env` | 客户 LLM key + 网关 token（**chmod 600**，红线） | 安装时 |
| `<out>/secrets.env` | WORKBENCH_TOKEN + GATEWAY_CLIENT_TOKEN（**chmod 600**） | install 生成 |
| `<out>/env.sh` | 成员交互环境（source 后直接 hermes -p <成员id>，**chmod 600**） | install 生成 |
| `<out>/config/config.json` | opc-init 产物（容器视角 /data 路径，勿直接改） | opc-init 生成 |
| `<out>/config/config.native.json` | 原生路径配置（/data → 宿主机绝对路径，install 自动重写） | install 生成 |
| `<out>/config/dispatcher.yaml` | dispatcher 配置（install 从模板生成） | install 生成 |
| `<out>/logs/*.log` | 采集/归因/网关同步日志 | */1、*/5 |

### 11.4 数据新鲜度快检（一条命令）

```bash
ls -la ~/opc-test/status.json ~/opc-test/token-stats.json   # mtime 距现在 < 2min 为正常
# 精确判定（分钟级）：
find ~/opc-test/status.json ~/opc-test/token-stats.json -mmin -2 -print   # 有输出 = 新鲜
```

采集周期速记：`*/1` opc_collect.py + token_stats.py、`*/5` llm_gateway_sync.py（crontab）；opc-bus 1s 轮询 kanban.db。
> 监控/告警成体系（v0.5.1 起）：告警总线（§11.5.1）+ 投递看门狗（§11.5.2）+ 代理守护（§11.5.3）+ 系统级监控（§11.5.4）已随部署包合入。

### 11.5 监控 / 告警 / 自愈组件（v0.5.1 合入，Gavin 0827「公司即产品」拍板）

> 各组件均来自 OPC 生产 08-27 事故实战（tg 投递链路抖动 / 降载误报 / 代理守护缺失 / 系统级监控缺失），
> 在生产验证通过后合入部署包。**路径参数化约定**：所有路径默认值指向部署用户 home，
> 一律支持环境变量覆盖（`OPC_*` / `WD_*` / `SCRIPT_DIR` 等），客户部署**必须**用 env 指向自己的目录，
> 严禁改脚本内硬编码（见 §14 红线 7）。

#### 11.5.1 告警总线（opc-alert-bus.py）

监控脚本（system-monitor / 看门狗 / server-monitor 等）在告警时同时 append 一条总线记录 →
路由 agent（cron 5min monitor 模式）读 `status=new` → 分级建 kanban 卡 → 卡 done 后 mark_resolved，
形成「告警 → 建卡 → 修复 → 闭环」自动链路。

```bash
# 冒烟（临时目录，零污染生产；OPC_ALERT_DIR 必须指向真实告警目录，生产为 <hermes-home>/opc/alerts）
export OPC_ALERT_DIR=/tmp/bus-smoke && mkdir -p "$OPC_ALERT_DIR"
python3 scripts/opc-alert-bus.py append --source system-monitor --severity P1 --type disk-root-high --title '根磁盘 ≥90%' --evidence 'df -h /: / 91%'
python3 scripts/opc-alert-bus.py read_new              # 看到刚 append 的 new 条目
python3 scripts/opc-alert-bus.py mark_resolved system-monitor:disk-root-high:$(date +%F)
# 单元自测（24 断言全绿）
python3 scripts/test_alert_bus.py
```

| CLI | 作用 |
|---|---|
| `append --source S --severity P0/P1/P2 --type T --title TTL [--evidence E]` | 写一条总线（P2 直接 resolved 信息留痕） |
| `read_new` / `count_new` | 未处理（status=new）条目输出/计数（路由 agent 用） |
| `mark_routed <fingerprint> <card_id>` / `mark_resolved <fingerprint>` | 建卡回写 / 闭环回写（幂等） |
| `list [--status ...]` | 调试用 |

- 总线文件：`<hermes-home>/opc/alerts/bus.jsonl`（O_APPEND 单行 JSON，schema 定稿 9 字段，勿改字段名）；
- fingerprint = `source:type:YYYY-MM-DD`（同日同源同类型幂等）；
- 路由 agent 挂载：`alerts_bus_monitor.py` 作 cron 5min monitor_script（输出稳定摘要，hash 不变 0 token 静默；
  未处理 age>2h 打 `[STALE>2h]` 自监控；18:50 汇总窗口供日报）。

#### 11.5.2 投递看门狗（telegram_delivery_watchdog.py）

检测外部投递链路（telegram 默认）异常：A. 日志滑动窗口错误统计（ConnectError / Timed out / blocked_config）；
B. jobs.json blocked_config 残留扫描。no_agent cron 契约：正常态 stdout 空（cron 静默不投递）、
异常态输出非空（cron 投递 → 看门狗输出触发告警总线自动建卡）。

```bash
# cron 挂载（30min）：output 为空 = 正常；非空 = 异常/恢复
*/30 * * * * /usr/bin/python3 /path/to/opc-os/scripts/telegram_delivery_watchdog.py
# 手动验证（正常态无输出）
python3 scripts/telegram_delivery_watchdog.py; echo "exit=$?"
```

- 阈值/路径全部 env 可覆盖：`WD_ERR24H_MAX`（24h 上限，默认 5）、`WD_ERR15M_MAX`（15min，默认 10）、
  `WD_ERR24H_ESC`（升级线 50，超线且错误数增长强制告警）、`WD_DEDUP_H`（去重窗口 2h）、
  `WD_STATE_FILE` / `WD_LOG_DIR` / `WD_JOBS`（状态/日志/jobs.json 路径）；
- 恢复：曾告警且回落阈值下 → 输出恢复行（P2 留痕）。

#### 11.5.3 代理守护（sing-box.service.in + singbox-watchdog.sh）

外部投递链路的代理底层守护（08-27 生产事故归因：代理节点抖动 → TG 长轮询超时 → 误报投递中断）。

```bash
# ① user unit 安装（前提：~/.local/bin/sing-box + ~/.config/sing-box/config.json 自备）
cp systemd/sing-box.service.in ~/.config/systemd/user/sing-box.service
systemctl --user daemon-reload && systemctl --user enable --now sing-box
systemctl --user is-active sing-box && systemctl --user show sing-box -p Restart   # active / Restart=always

# ② 自愈看门狗（cron 1min；stdout 只在状态变化时输出，兼容 no_agent 静默契约）
* * * * * /bin/bash /path/to/opc-os/scripts/singbox-watchdog.sh
```

- 自愈阶梯：L1 SIGHUP 重载 → L2 重启 sing-box → L3 熔断（30min 窗口 ≥3 次重启后冷却 15min）；
- v2.1 保护：telegram 非 connected 需持续 ≥5min（`GW_RETRY_WINDOW`）才动 gateway，防瞬态误杀；
  近 10min 有运行中任务（`RUNNING_JOB_WINDOW`）推迟重启；
- 路径 env 可覆盖：`SCRIPT_DIR` / `LOG_FILE` / `GW_STATE_JSON` / `EXECUTIONS_DB`；
- 无 systemd 环境：unit 跳过，watchdog nohup 守护覆盖重启。

#### 11.5.4 系统级监控（system-monitor.py，v0.5.1 评估 · 2026-08-28 合入）

系统级统一监控：每 5 分钟采集负载/内存/swap/磁盘/tmp/温度/OPCOS 服务/端口/网络速率/TCP/探针，
追加一条留底记录（`sysmon.jsonl`），异常时触发自救（清 /tmp、截日志、停可杀服务）+ 输出告警，
整点输出每小时汇总；降载感知（P0 复验 PASS 后合入）：受控降载停服/停端口在登记期间不算故障（防降载误报），
降载必须可逆（`heal_restore` 恢复），降载闭环经告警总线 P2 `degrade` 留痕。

```bash
# cron 挂载（5min；no_agent 纯脚本：stdout 空=静默不投递，非空=告警/整点汇总）
*/5 * * * * /usr/bin/python3 /path/to/opc-os/scripts/system-monitor.py
# 手动冒烟（临时目录，零污染生产；OPC_* 指向临时路径后跑一次，验证可执行 + 不触碰生产）
export OPC_SYSMON_DIR=/tmp/sysmon-smoke/state OPC_SYSMON_LOG=/tmp/sysmon-smoke/system-health.log
export OPC_ALERT_DIR=/tmp/sysmon-smoke/alerts
mkdir -p "$OPC_SYSMON_DIR" "$OPC_ALERT_DIR"
python3 scripts/system-monitor.py; echo "exit=$?"    # 有真实异常会输出（只读、写临时），无则静默
# 幂等单元自测（19 项全绿；路径经 __file__ 解析 + 临时 OPC_ALERT_DIR，生产/产品包两处可跑、零污染）
python3 scripts/.system-health/test_degrade_perception.py
```

- **env 全参数化**（默认值=生产路径兜底，零行为变化；客户部署必须用 env 指向自己的目录，见 §14 红线 7）：
  `OPC_HOME`（base 目录）、`OPC_SYSMON_DIR`（状态目录→net_state.json/sysmon.jsonl/selfheal_state.json/.last-boot-id/w401_state.json）、
  `OPC_SYSMON_LOG`（文本日志）、`OPC_KANBAN_DB` / `OPC_KANBAN_LOCK` / `OPC_GATEWAY_LOG`（kanban 调度健康观测）、
  `OPC_DISPATCHER_STATE` / `OPC_HERMES_CFG` / `OPC_PROFILES_DIR` / `OPC_XDG_RUNTIME_DIR`（调度/环境自检）、
  `OPC_BUS_SCRIPT`（告警总线脚本，默认 `<home>/.hermes/scripts/opc-alert-bus.py`）；
- 输出契约：stdout 只在「有告警」或「整点汇总」时非空；纯采集时静默（兼容 cron no_agent）；
- 自愈要点：受控降载名单 `HEAL_STOP_SERVICES` / 端口映射 `HEAL_SERVICE_PORTS` 为可逆登记（恢复阈值温度<80 + 可用内存>20% 滞回防抖）；
- 依赖：告警总线（§11.5.1）已在包内（`_bus_append` 子进程调用），无第三方 Python 库。

### 11.6 EnvironmentFile 加载核查（token 注入）

**背景**：token（GATEWAY_CLIENT_TOKEN / WORKBENCH_TOKEN）不写进 unit，统一经 `EnvironmentFile` 注入（5 个 `opcos-*.service` 各 1 行，§8.8）。出现 401 / 成员对话失败时，**先查注入面再查服务本身**。

```bash
# ① 注入面存活（服务实际加载的 env 文件）
systemctl show opcos-dispatcher -p EnvironmentFiles    # EnvironmentFiles=/path/secrets.env (ignore_errors=no)

# ② 注入面 token 有值
EF=$(systemctl show opcos-dispatcher -p EnvironmentFiles | cut -d= -f2- | cut -d' ' -f1)
grep -c '^GATEWAY_CLIENT_TOKEN=.' "$EF"               # 1

# ③ 告警信号：systemd 找不到 env 文件时记 Failed to load environment files
journalctl -u opcos-* --since today | grep -i 'EnvironmentFile\|Failed to load' || echo clean
```

---

## 12. 备份与恢复（backup.sh）

> 以当前 `scripts/backup.sh` **实际行为**为准（复用 mrd_backup.sh 模式：成功静默、失败输出错误并中止）。
> v0.6 起备份集由 4 → **7 文件**（GAP-6 P1-1 代码部分随升级机制合入）。

### 12.1 备份（当前行为）

```bash
# 原生形态（推荐）：从本地数据目录备份
cd ~/opc-os && scripts/backup.sh --data-dir ~/opc-test
# 结果：<备份根>/<日期>/ 下 7 个文件

# 自定义保留份数 / 备份根目录
scripts/backup.sh --data-dir ~/opc-test --keep 30          # 保留最近 30 份（默认 14）
scripts/backup.sh --data-dir ~/opc-test --out /mnt/ops     # 备份到 /mnt/ops/<日期>/
```

要点（实测）：
- **备份集 = 7 文件**，分两级：
  - **核心 4 文件（缺一即失败中止）**：`kanban.db`（任务/事件留痕）+ `status.json` + `token-stats.json` + `state.json`；
  - **配置类 3 文件（缺 = 警告继续，新部署可能尚无）**：`gateway.config.json`（网关运行时配置）+ `bus-state.json`（数据总线水位线）+ `govern-state.json`（治理状态）；
- **核心缺一即失败**：`--data-dir` 目录缺任一核心文件 → 清理本次已复制部分 + 报错退出（防不完整备份被误当完整）；
- **默认保留最近 14 份**（按日期目录 mtime 裁剪，`--keep` 调整）；
- **env 参数化**（§14 红线 7）：`OPC_DATA_DIR`（数据目录兜底，等价 `--data-dir`，默认空）、`OPC_GATEWAY_CONFIG`
  （网关配置路径，默认 `<data-dir>/llm-gateway/gateway.config.json`，生产兜底 `~/.hermes/opc/llm-gateway/gateway.config.json`）；
- 兼容分支：未给 `--data-dir` 时尝试从 docker 容器 `opc-os-collector:/data` 导出（v0.2 旧形态部署兼容；**原生形态请显式 --data-dir**）；
- 备份**不含** llm.env / secrets.env / hermes-home（凭据与成员配置不随备份流转）。

建议纳入日常 cron（deploy-guide §4 同款）：

```bash
# 每天 03:30 备份
30 3 * * * cd /path/to/opc-os && scripts/backup.sh --data-dir ~/opc-test >> backups/backup.log 2>&1
```

### 12.2 恢复步骤

```bash
# ① 找到要恢复的备份
ls ~/opc-os/backups/                                    # 按日期目录列出

# ② 停栈（避免运行中写库与恢复互相干扰；推荐）
scripts/install-services.sh stop --out ~/opc-test

# ③ 恢复前保护现场（红线：覆盖前先备份当前状态）
cp ~/opc-test/kanban.db ~/opc-test/kanban.db.bak-$(date +%F)

# ④ 恢复核心 4 个数据文件
cp ~/opc-os/backups/<日期>/kanban.db ~/opc-test/
cp ~/opc-os/backups/<日期>/status.json ~/opc-test/
cp ~/opc-os/backups/<日期>/token-stats.json ~/opc-test/
cp ~/opc-os/backups/<日期>/state.json ~/opc-test/

# ④b 恢复配置类 3 文件（视部署形态落点；网关配置默认 <data-dir>/llm-gateway/）
cp ~/opc-os/backups/<日期>/gateway.config.json ~/opc-test/llm-gateway/
# bus-state.json / govern-state.json 复制到该部署的相应数据落点

# ⑤ 起栈 + 验证
scripts/install-services.sh start --out ~/opc-test
scripts/smoke_test.sh 8081                               # 37/37 ALL GREEN；页面数据恢复
```

> 恢复不动 hermes-home / llm.env / secrets.env（备份集本就不含）——成员配置与凭据保持现状。
> 恢复后 opc-bus 会按新 kanban.db 重算 kanban 段（水位线 bus-state.json 若超前会被安全跳过，无需手工处理）。
> **v0.6 变更**：备份集已由 4 → 7 文件（+gateway.config.json / bus-state.json / govern-state.json，GAP-6 P1-1 代码部分已合入）。
> 「备份 → 删数据 → 恢复 → 数据一致」演练见 delivery-checklist §2 验收点。

---

## 13. 常见坑（本次实测全记录）

| # | 坑 | 现象 | 解决 |
|---|---|---|---|
| 1 | **10.0.2.2 代理假死** | 用 10.0.2.2:7890 测代理 000，误判「无需代理」 | 正确地址 10.77.0.1:7890（沙箱内网桥）；别用宿主侧地址测连通性 |
| 2 | **Ubuntu 24.04 缺 python3-venv** | install-hermes-worker.sh venv 创建失败 rc=1 | `sudo apt install -y python3.12-venv` 后重跑 |
| 3 | **apt 装 node 是半套** | node v18.19.1 且无 npm | 走 nodesource setup_22.x 装 LTS（v22 + npm 10） |
| 4 | **成员对话 401（GATEWAY_CLIENT_TOKEN 未注入）★ 本次新坑** | `hermes -p dev chat` 报 AuthenticationError [HTTP 401]（重试 3 次，看起来像卡死）；根因：profile 的 `api_key=${GATEWAY_CLIENT_TOKEN}` 是 env 引用，shell 未导出时为字面量 | `source ~/opc-test/env.sh` 再跑（脚本自动生成 env.sh）；dispatcher 侧由 secrets.env EnvironmentFile 自动注入 |
| 5 | **kanban 卡不派活（skipped_nonspawnable=N）★ 本次新坑** | 卡建好一直 ready，dispatcher 日志 `skipped_nonspawnable` 持续增长、`spawned=0`；根因：dispatcher 进程缺 **HERMES_HOME** env，dispatch_once 用 resolve_profile_env 校验 assignee profile 有效性时解析到 ~/.hermes/profiles（不存在）判为 nonspawnable | install-services.sh 已在 dispatcher unit 注入 `Environment=HERMES_HOME=<out>/hermes-home`（v0.4 已修复，重装即生效） |
| 6 | **enable --now 不重启已 active 服务 ★ 本次新坑** | 改了 unit 模板重跑 install，服务还是旧配置（systemctl enable --now 对已在跑的服务幂等跳过） | install-services.sh 的 start 语义已改为 `enable + restart`（v0.4 已修复） |
| 7 | **config.json /data 路径（容器视角）★ 本次新坑** | opc-init 产物 paths 全 `/data/...`，原生进程直接读会写错目录 | install-services.sh 自动生成 `config.native.json`（/data → <out>，json 级安全替换，实测 9 项） |
| 8 | **空 CAPACITY env 噪音** | 未设 --mem-mb/--cpus 时 dispatcher 日志每条 tick 打 WARNING（`invalid ... '' ignored`），无害但刷屏 | render 时已删空值行（v0.4 已修复） |
| 9 | **LXC 无 systemd** | 精简 LXC 没有 /run/systemd/system，systemctl 不可用 | install-services.sh 自动检测退化 nohup+pidfile（`start/stop/status`）；也可 `--no-systemd` 强制 |
| 10 | **首装假失败（历史）** | 校验段 head -1 SIGPIPE 致首装误判失败 | 已修复；重跑时看脚本退出码，勿跳过校验段 |

---

## 14. 红线（违反即验收不通过）

1. **凭据不落盘**：上游 LLM key 只进 `<out>/llm.env`（chmod 600）；`secrets.env` 只含 WORKBENCH_TOKEN + GATEWAY_CLIENT_TOKEN（600）；文档/代码/聊天零明文 key，一律 `<TOKEN>` 占位。
2. **脱敏**：测试公司名/成员名虚构（如「北辰软件」），禁止任何真实团队身份。
3. **外部访问统一走代理**：10.77.0.1:7890（已实测），勿用 10.0.2.2。
4. **config.yaml 一律官方命令写**：不手写、不 sed（install 脚本已内置该红线）。
5. **运行时零外连**：构建完成后整套系统运行不需要外部网络（页面只读本地 JSON）。
6. **systemd unit 是部署机运行时配置**（/etc/systemd/system/opcos-*.service 由模板渲染，不入代码仓明文 key）。
7. **监控/告警/守护组件路径参数化（v0.5.1 起）**：opc-alert-bus.py / alerts_bus_monitor.py / telegram_delivery_watchdog.py / singbox-watchdog.sh / system-monitor.py（§11.5.4）一律经 `OPC_*` / `WD_*` / `SCRIPT_DIR` 等环境变量指向部署目录（默认值仅为生产兜底 `/home/gavin`）；**禁止**在客户化时改脚本内硬编码路径（部署用 env 覆盖，§11.5）。新增监控组件合入前必须带 env 参数化 + 自测。

---

## 附：本次实测环境与耗时速查

| 步骤 | 版本/结果 | 耗时 |
|---|---|---|
| nodesource 装 node LTS | node v22.23.2 / npm 10.9.8 | ~85s |
| uv 官方脚本 | uv 0.12.5 | ~8s |
| rsync opc-os + hermes-agent-src | 双仓推送 | ~3min |
| opc-init 参数化（software 模板） | 9 角色产物 + llm.env(600) | ~1s |
| install-hermes-worker.sh | fork venv + 9 profiles（见 §6） | 见 §6 |
| **install-services.sh install** | **5 服务 systemd 启动（重跑幂等）** | **~3.3s** |
| 网关 /v1/models + 真实调用 | HTTP 200 / 200 | 0.92s |
| 成员走网关（dev 对话） | 经网关真实回复 | ~4s |
| kanban 建卡 → 完成 | ready → running → done（含 60s tick） | ~2.5min |
| smoke_test.sh | **37/37 ALL GREEN** | ~10s |
| backup.sh --data-dir | kanban.db/status/token-stats/state → backups/<日期>/ | ~1s |