# OPC OS 安装包

## 版本
- 当前版本：**v0.7.0**（安装包 `opc-os-v0.7.0.tar.gz`）

## 下载
- 安装包：`opc-os-v0.7.0.tar.gz`
- 安装文档：`installation-guide.md`（从零到可用完整手册，已实测）

## 安装前提
- node >= 20 + python3.10+（含 uv）
- 已去 docker 化，5 个服务为原生进程（systemd 守护；无 systemd 自动退化 nohup）
- LXC 容器也可运行

## 快速开始（三步）
```bash
# 解压
tar -xzf opc-os-v0.7.0.tar.gz
cd opc-os

# ① 生成客户团队（参数化，非交互）
python3 scripts/opc_init.py --template software --name "<客户公司名>" --gateway --out ~/opc-test

# ② 安装 Hermes 团队运行时（fork 底座 + 每成员 profile 走网关）
bash scripts/install-hermes-worker.sh \
  --source ~/hermes-agent-src \
  --hermes-home ~/opc-test/hermes-home \
  --gateway-url http://127.0.0.1:18080/v1

# ③ 起栈（5 服务 systemd 守护）
scripts/install-services.sh install --out ~/opc-test
```

详见 `installation-guide.md`。
