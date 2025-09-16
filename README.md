K8s Ops Lab（Mac M4 + Parallels + Ubuntu 24.04）

目的：在本地 Mac（M4）+ Parallels 的 Ubuntu 24.04 环境中，搭一个可复现、可演示、可扩展的 Kubernetes 运维实验集群，练习生命周期、GitOps、Observability、安全基线与故障演练。

架构概览

宿主：macOS → Parallels → Ubuntu 24.04（ARM64）

集群：Kind（多节点）

Ingress：ingress-nginx（NodePort 映射到 VM 80/443）

应用：nginxdemos/hello（后续可替换为自建 Java/Spring Boot）

可观测性：kube-prometheus-stack（Prometheus/Alertmanager/Grafana）、Loki

GitOps：Argo CD（可选启用）
