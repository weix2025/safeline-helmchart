# SafeLine WAF Helm Chart

基于 [chaitin/safeline](https://github.com/chaitin/safeline) 的 Kubernetes Helm Chart。

> 原始项目: [yaencn/safeline-helmchart](https://github.com/yaencn/safeline-helmchart)
> 本版本 (10.2.0) 经过安全加固和最佳实践优化，适配 k3s + Longhorn + MetalLB 环境。

## 环境要求

- Kubernetes 1.22+ / k3s
- Helm 3.x
- 存储: Longhorn（推荐）或其他支持 RWO/RWX 的 StorageClass
- 负载均衡: MetalLB（推荐）或 NodePort

## 快速开始

```bash
# 从本地目录安装
helm install safeline ./safeline/charts \
  --namespace safeline \
  --create-namespace \
  --set database.internal.password="你的强密码"

# 从远程仓库安装
helm repo add yaencn https://helm.yaencn.com/charts
helm install safeline yaencn/safeline \
  --namespace safeline \
  --create-namespace

# 升级
helm -n safeline upgrade safeline ./safeline/charts

# 卸载
helm -n safeline uninstall safeline
```

## 生产部署示例

### MetalLB + Longhorn 环境

```bash
helm install safeline ./safeline/charts \
  --namespace safeline \
  --create-namespace \
  --set database.internal.password="$(openssl rand -base64 24)" \
  --set tengine.service.type=LoadBalancer \
  --set tengine.service.loadBalancerIP="192.168.1.100" \
  --set persistence.persistentVolumeClaim.database.storageClass=longhorn \
  --set persistence.persistentVolumeClaim.nginx.storageClass=longhorn \
  --set persistence.persistentVolumeClaim.mgt.storageClass=longhorn
```

### 通过 Ingress 访问管理控制台

```yaml
global:
  ingress:
    enabled: true
    hostname: waf.example.com
    ingressClassName: nginx
    tls:
      secretName: "waf-example-com-tls"
```

## 核心配置参数

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `global.image.registry` | 镜像仓库地址 | `swr.cn-east-3.myhuaweicloud.com/chaitin-safeline` |
| `global.timezone` | 时区 | `Asia/Shanghai` |
| `global.busyboxImage` | initContainer 基础镜像 | `busybox:1.36` |
| `global.exposeServicesAsPorts.enabled` | 以端口模式暴露服务（建议开启） | `true` |
| `database.internal.password` | 数据库密码（**必须修改**） | `changeit` |
| `database.type` | 数据库类型 `internal`/`external` | `internal` |
| `tengine.service.type` | tengine 服务类型 | `LoadBalancer` |
| `tengine.service.loadBalancerIP` | MetalLB 指定 IP | `""` |
| `mgt.service.type` | 管理控制台服务类型 | `NodePort` |
| `mgt.service.web.nodePort` | 管理控制台 NodePort 端口 | `31443` |
| `persistence.persistentVolumeClaim.*.storageClass` | 存储类 | `""` (使用默认) |

## 架构组件

| 组件 | 说明 | 端口 |
|------|------|------|
| **tengine** | WAF 反向代理引擎，处理入站流量 | 80, 65443, 9999 |
| **detector** | 威胁检测引擎，语义分析 | 8000, 8001, 7777 |
| **mgt** | 管理控制台 Web UI | 1443, 80, 8000 |
| **chaos** | 人机验证 / 等候室 | 9000, 23333, 8080 |
| **fvm** | 特征/漏洞管理 | 9004, 80 |
| **luigi** | 日志处理 | 80 |
| **database** | PostgreSQL 数据库 | 5432 |

## 安全注意事项

1. **数据库密码**: 默认密码为 `changeit`，**必须**在部署前修改
2. **数据库连接串**: 已从 ConfigMap 迁移到 Secret 存储，避免明文泄露
3. **JWT 密钥**: `chaos-cm.yaml` 中包含默认 EC 私钥，生产环境建议替换：
   ```bash
   openssl ecparam -genkey -name prime256v1 -noout -out ec_private.pem
   openssl ec -in ec_private.pem -pubout -out ec_public.pem
   ```
4. **副本数限制**: 所有 Deployment 只能运行 **1 个副本**，多副本会导致 WAF 服务异常

## 多 IDC 部署

每个 IDC 机房独立部署一套 Helm release，通过 values 文件区分：

```bash
# IDC-A
helm install safeline-a ./safeline/charts -f values-idc-a.yaml -n safeline

# IDC-B
helm install safeline-b ./safeline/charts -f values-idc-b.yaml -n safeline
```

各 IDC 的 `values-idc-x.yaml` 只需覆盖差异项：

```yaml
tengine:
  service:
    loadBalancerIP: "10.0.1.100"  # 该机房的入口 IP
persistence:
  persistentVolumeClaim:
    database:
      storageClass: "local-path"  # 按机房存储后端调整
```

## 国际版部署

从 appVersion 8.8.2 开始支持 x86_64 架构的国际版：

```bash
helm install safeline ./safeline/charts \
  --set global.image.registry=chaitin \
  --set global.image.region="-g"
```

## v10.2.0 变更日志（基于 10.1.0 审查优化）

### 安全加固
- 默认密码 `changeit` 现在会在 `helm install` 时直接报错，强制用户设置强密码
- `database.external.password` 添加 `required` 校验

### Bug 修复
- 修复 `database-ss.yaml` initContainer 引用不存在的 `$database.subPath`
- 修复 `pg_isready` 探针硬编码 `safeline-ce`，改用模板函数动态生成
- 修复 `mgt-dpl.yaml` 在 `persistence.enabled=false` 时缺少 `nginx-log` 和 `safeline-sock-dir` volume
- 修复 `tengine-svc.yaml` NodePort 模式下 `httpNodePort`/`healthNodePort` 未映射到模板
- 修复 Ingress 资源名称硬编码 `waf-dashboard-domain-ingress`，改为跟随 release name
- 修复 `database-url-secret.yaml` 在 `existingSecret` 已配置时仍重复创建 Secret
- 修复 `readinessProbe.initialDelaySeconds` 从 30s 降为 5s（数据库就绪探测不应等 30s）

### 其他
- `appVersion` 从 `9.3.2` 更正为 `10.1.0`
- NOTES.txt 拼写修正 `managenment` → `management`

## v10.1.0 变更日志（基于 10.0.32 优化）

### 安全加固
- 数据库连接串从 ConfigMap 迁移到 Secret 存储
- 添加默认密码修改提醒
- 添加 JWT 密钥替换提醒

### Bug 修复
- 修复数据库 Service 名称硬编码 `safeline-pg`，现跟随 release name
- 修复 mgt/fvm/luigi Service 类型 `LoadBalancer` 被强制降级为 `NodePort`
- 修复 ConfigMap 名称硬编码导致多实例冲突
- 修复 `database-ss.yaml` 引用不存在的 `subPath`

### 最佳实践
- 为所有 Deployment 添加 `livenessProbe`
- 替换 `hostPath /etc/localtime` 为 `TZ` 环境变量
- initContainer busybox 镜像改为可配置
- 更新策略改为 `maxSurge:1, maxUnavailable:0`（零停机更新）
- 清理未使用的 `startupProbe` 死代码
- Chart apiVersion 升级到 v2（Helm 3 标准）

### 新增功能
- tengine Service 支持 LoadBalancer 类型（配合 MetalLB）
- 新增 `global.timezone` 全局时区配置
- 新增 `global.busyboxImage` 全局基础镜像配置

## 打包与发布

```bash
# 验证 Chart 语法
helm lint ./safeline/charts

# 打包为 tgz
helm package ./safeline/charts -d ./assets/safeline/

# 生成/更新 Helm 仓库索引
helm repo index ./assets/safeline/ --url https://helm.yaencn.com/charts
```

## License

MIT
