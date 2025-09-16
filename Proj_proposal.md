# 仓库名称建议
**k8s-ops-lab**（可选：`k8s-ops-lab-mac-parallels`）

---

# 工作要求（Working Agreements）
**目标导向**：每个任务以“可验证的产出”为中心（截图、命令输出、指标图或演示链接）。

**开发流程**：
- **分支策略**：
  - `main`：稳定分支（只允许通过 PR 合并，需通过 CI）。
  - `feat/*`、`fix/*`、`chore/*`：功能/修复/杂项分支。
- **提交规范（Conventional Commits）**：`feat(scope): ...`、`fix(scope): ...`、`docs(scope): ...` 等；scope 建议：`cluster|apps|ingress|gitops|monitoring|logging|security|ci|docs`。
- **PR 要求**：
  - 关联 Issue（`Closes #<id>`）
  - 勾选检查清单（见 PR 模板）
  - 附关键截图（Grafana/Dashboard、k9s、kubectl 输出）
  - 如改动运行清单（YAML/Helm/Kustomize），附 `diff` 或说明

**验收标准（Definition of Done）**：
- 文档：README 或 `runbooks/` 更新到位
- 自动化：Makefile/脚本更新并可重复执行
- 观测：若功能影响运行路径，提供指标/日志截图证明
- 安全：RBAC/NetworkPolicy/镜像扫描（如适用）

**标签体系（Labels）**：
- `type/feat|fix|docs|refactor|ci|test|ops`
- `area/cluster|apps|ingress|gitops|monitoring|logging|security|backup`
- `priority/p0|p1|p2`（p0=阻断）
- `size/XS|S|M|L`（预估：≤30m、≤2h、≤0.5d、≤1d+）
- `status/blocked|need-info|ready-for-review`

**节奏与追踪**：
- 每日 `runbooks/notes-YYYYMMDD.md` 简短记录：What/Blocked/Next
- 每完成一个里程碑，提交 1 份“复盘（postmortem）”摘要（问题清单&改善项）

---

# README（可直接用作仓库首页）

## K8s Ops Lab（Mac M4 + Parallels + Ubuntu 24.04）
**目的**：在本地 Mac（M4）+ Parallels 的 Ubuntu 24.04 环境中，搭一个**可复现、可演示、可扩展**的 Kubernetes 运维实验集群，练习生命周期、GitOps、Observability、安全基线与故障演练。

### 架构概览
- 宿主：macOS → Parallels → Ubuntu 24.04（ARM64）
- 集群：Kind（多节点）
- Ingress：ingress-nginx（NodePort 映射到 VM 80/443）
- 应用：`nginxdemos/hello`（后续可替换为自建 Java/Spring Boot）
- 可观测性：kube-prometheus-stack（Prometheus/Alertmanager/Grafana）、Loki
- GitOps：Argo CD（可选启用）

![architecture](./docs/architecture.drawio) <!-- 预留：后续导入图 -->

---

## 快速开始（Quick Start）
### 1) 先决条件
- Parallels 安装 Ubuntu 24.04（ARM64）VM，推荐 **4 vCPU / 12–16GB RAM / 80GB 磁盘**
- VM 网络建议 **Shared**（10.211.55.x）；若使用 VPN，参考文档“网络与访问”
- Ubuntu 内安装：Docker、kind、kubectl、helm、k9s、make、curl、git

### 2) 克隆与初始化
```bash
git clone https://github.com/<you>/k8s-ops-lab.git
cd k8s-ops-lab
```

### 3) 创建集群与 Ingress
```bash
make cluster
make ingress
```
> `make` 命令见 [Makefile 目标](#makefile-目标)

### 4) 部署 Demo 应用
```bash
make hello
```

### 5) 从 Mac 访问
- 获取 VM IP（Ubuntu 内）：`hostname -I | awk '{print $1}'`
- 打开浏览器：`http://hello.<VM_IP_点换成->.nip.io`  
  例：VM_IP=10.211.55.20 → `http://hello.10-211-55-20.nip.io`

---

## 仓库结构
```
.
├─ cluster/
│   ├─ kind.yaml              # 默认集群（含端口映射）
│   └─ kind-netpol.yaml       # 禁用默认 CNI（供 NetworkPolicy 演练）
├─ apps/
│   └─ hello/
│       ├─ Dockerfile         # （后续自建服务使用）
│       └─ k8s/
│           └─ base/          # 最小可用 YAML（NS/Deployment/Service/Ingress/HPA）
├─ gitops/
│   └─ argocd/                # Argo CD 安装与应用清单（可选）
├─ ops/
│   ├─ monitoring/            # kube-prometheus-stack 等
│   └─ logging/               # loki/promtail
├─ runbooks/
│   ├─ 01-app-down.md
│   ├─ 02-node-notready.md
│   └─ 03-disk-full.md
├─ docs/
│   ├─ architecture.drawio
│   └─ screenshots/
├─ .github/
│   ├─ workflows/
│   │   └─ ci.yaml            # CI: 构建镜像、YAML 检查、镜像扫描
│   └─ ISSUE_TEMPLATE/
│       ├─ bug_report.md
│       └─ task.md
├─ Makefile
└─ README.md
```

---

## Makefile 目标
```Makefile
CLUSTER?=opslab

cluster:
	kind create cluster --config cluster/kind.yaml --name $(CLUSTER)
	kubectl cluster-info

ingress:
	helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
	helm repo update
	helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
	  -n ingress-nginx --create-namespace \
	  --set controller.service.type=NodePort \
	  --set controller.service.nodePorts.http=30080 \
	  --set controller.service.nodePorts.https=30443

hello:
	kubectl apply -f apps/hello/k8s/base

clean:
	kind delete cluster --name $(CLUSTER)
```

---

## 网络与访问（VPN/Shared 模式下的常见做法）
- **nip.io 域名**：`hello.<10-211-55-XX>.nip.io` 指向 VM IP，便于浏览器访问
- **路由法（Mac）**：`sudo route -n add -net 10.211.55.0/24 -interface vnic0`
- **端口转发法（Parallels）**：把 Mac 的 8080→VM:80，8443→VM:443
- **SSH 转发**：`ssh -N -L 8080:127.0.0.1:80 ubuntu@<VM_IP>`

---

## 可观测性（Prom/Grafana/Loki）
```bash
# 监控栈
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm upgrade --install kube-prom-stack prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace

# 日志（可选 Loki）
helm repo add grafana https://grafana.github.io/helm-charts
helm upgrade --install loki grafana/loki -n logging --create-namespace
helm upgrade --install promtail grafana/promtail -n logging --create-namespace

# 访问 Grafana
kubectl -n monitoring port-forward svc/grafana 3000:80
# 浏览器： http://localhost:3000 （默认 admin / prom-operator 或按 chart 版本查看 secret）
```

---

## GitOps（可选）
```bash
# 安装 Argo CD
kubectl create ns argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
# 访问 UI
kubectl -n argocd port-forward svc/argocd-server 8081:80
```
将 `gitops/argocd/apps/hello.yaml` 指向 `apps/hello/k8s/overlays/*`，实现 Git 驱动的部署。

---

## 安全基线（NetworkPolicy / CNI）
> 需用 `cluster/kind-netpol.yaml` 重建集群，并安装 Cilium/Calico 才能生效。
```bash
kind delete cluster --name opslab
kind create cluster --config cluster/kind-netpol.yaml --name opslab
helm repo add cilium https://helm.cilium.io/
helm upgrade --install cilium cilium/cilium -n kube-system \
  --set kubeProxyReplacement=strict \
  --set k8sServiceHost=$(kubectl get node -l node-role.kubernetes.io/control-plane -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}') \
  --set k8sServicePort=6443
```
随后添加：
- Pod Security Admission 注解（`restricted`）
- 逐步放开 `NetworkPolicy`（默认 deny all → 服务化放通）
- CI 引入 Trivy 镜像扫描

---

## 故障排查（Runbooks）
- Pod CrashLoopBackOff（镜像名/探针错误）
- 节点维护（cordon/drain）
- 网络策略误配导致 5xx
- 资源配额触发导致部署失败
- Ingress 主机名/证书问题

每个场景提供：背景→症状→诊断路径（命令/看板）→修复步骤→后续改进。

---

## CI（示例 `.github/workflows/ci.yaml` 片段）
```yaml
name: ci
on:
  push:
    branches: [ main ]
    paths: [ 'apps/**', 'cluster/**', 'ops/**', 'Makefile', 'README.md' ]
  pull_request:
    branches: [ main ]

jobs:
  lint-yaml:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ibiqlik/action-yamllint@v3

  build-multiarch:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-qemu-action@v3
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          context: apps/hello
          push: false # 初期不推注册表，验证能构建
          platforms: linux/amd64,linux/arm64

  trivy-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aquasecurity/trivy-action@0.24.0
        with:
          scan-type: 'fs'
          severity: 'CRITICAL,HIGH'
```

---

## 许可证
MIT（可根据需要调整）

---

# 里程碑（Milestones）与计划节点（按 PST / America/Los_Angeles）
> 今日为 **2025-09-16**。

### M0｜仓库初始化 & 连通性（截止：2025-09-17）
- ✅ 建立仓库、推送 README、Makefile、最小清单
- ✅ 在 VM 内可访问 `curl http://localhost`，Mac 端通过 nip.io/端口转发访问成功
- Issues：#1 仓库脚手架、#2 Parallels 端口转发、#3 nip.io 域名访问验证

### M1｜Phase 0 快速起跑（截止：2025-09-18）
- ✅ kind 集群 + ingress-nginx + hello 应用
- ✅ README“快速开始”可复现（新机器 1 小时内跑通）
- Issues：#4 部署 hello、#5 Ingress 校验、#6 HPA 最小示例

### M2｜生命周期 & 调度（截止：2025-09-22）
- ✅ cordon/drain 演练，滚动升级演示
- ✅ ResourceQuota + LimitRange；affinity/taints 演示
- Issues：#7 调度策略样例、#8 资源配额、#9 升级回滚 Runbook

### M3｜CI/CD + GitOps（截止：2025-09-26）
- ✅ GitHub Actions 能构建多架构镜像
- ✅ Argo CD 同步 `apps/hello`（dev→prod overlay）
- Issues：#10 CI 工作流、#11 推送 GHCR、#12 Argo 应用清单

### M4｜Observability（截止：2025-09-30）
- ✅ kube-prometheus-stack + 常用看板
- ✅ Loki + promtail 查询应用日志
- Issues：#13 安装监控、#14 告警规则、#15 日志聚合

### M5｜安全基线 & NetPol（截止：2025-10-03）
- ✅ 重建启用 Cilium；默认 deny all → 精细放通
- ✅ Pod Security Admission（restricted）
- Issues：#16 CNI 安装、#17 PSP→PSA 注解、#18 NetPol 策略集

### M6｜可靠性 & 有状态（截止：2025-10-06）
- ✅ HPA+VPA 基本实践；PDB 与优雅终止
- ✅ local-path-provisioner 或 NFS 作为存储（可选）
- Issues：#19 VPA 演示、#20 PDB/终止钩子、#21 存储类

### M7｜备份恢复（可选）（截止：2025-10-08）
- ✅ MinIO + Velero：备份/恢复/跨集群迁移演示
- Issues：#22 部署 Velero、#23 备份策略、#24 恢复演示

### M8｜Break & Fix + 成果包装（截止：2025-10-10）
- ✅ 至少 3 个故障演练 Runbook 完整闭环
- ✅ README 截图齐全 + 架构图 + 小结
- Issues：#25 故障注入脚本、#26 复盘模板、#27 README 完成度检查

---

# Issue / PR 模板

**`.github/ISSUE_TEMPLATE/bug_report.md`**
```markdown
---
name: Bug report
about: 报告问题
labels: type/bug, priority/p1
---

**现象**
-
**期望**
-
**复现步骤**
1.
2.
**截图/日志**
-
**环境**
- VM IP / Cluster name / 相关组件版本
```

**`.github/ISSUE_TEMPLATE/task.md`**
```markdown
---
name: Task
about: 可交付的小任务
labels: type/ops
---

**目标（Definition of Done）**
- [ ]
**实施要点**
-
**验收证明**（截图/命令输出/链接）
-
```

**`.github/pull_request_template.md`**
```markdown
## 变更内容
-

## 关联 Issue
Closes #

## 检查清单
- [ ] 通过 CI
- [ ] 更新 README/Runbook（如适用）
- [ ] 附关键截图或命令输出
- [ ] 安全基线检查（RBAC/NetPol/镜像扫描）（如适用）
```

---

# 下一步
1. 在 GitHub 新建空仓库（私有/公开均可）：`k8s-ops-lab`
2. 本地初始化并推送：
   ```bash
   git init
   git remote add origin git@github.com:<you>/k8s-ops-lab.git
   git add .
   git commit -m "feat(repo): scaffold k8s-ops-lab"
   git push -u origin main
   ```
3. 在仓库 Settings → Actions 启用 GitHub Actions；在 Issues 里创建里程碑（按上文 M0~M8）。
4. 开 `issue #1`：仓库脚手架对齐（目录、Makefile、清单与 README）。

