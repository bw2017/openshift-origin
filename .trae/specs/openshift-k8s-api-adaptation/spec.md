# OpenShift Kubernetes API 适配规范

## Why
OpenShift Origin 当前基于 Kubernetes v1.11.0，现已升级到 v1.35.3。从 v1.11 到 v1.35 跨越 24 个版本，Kubernetes 对内部 API 包结构进行了重大重组，导致 OpenShift 代码中的 import 路径大量失效。

## What Changes

### 1. API 包路径变更（BREAKING）
Kubernetes 1.35 将大量内部 API 从 `k8s.io/kubernetes/pkg/*` 移到了 staging 目录：

| 旧路径 (v1.11) | 新路径 (v1.35) |
|----------------|-----------------|
| `k8s.io/kubernetes/pkg/api/legacyscheme` | `k8s.io/kubernetes/staging/src/k8s.io/api` → `k8s.io/api` |
| `k8s.io/kubernetes/pkg/apis/*` | `k8s.io/api/*/v1` |
| `k8s.io/kubernetes/pkg/master` | `k8s.io/kubernetes/staging/src/k8s.io/apiserver/pkg/master` |
| `k8s.io/kubernetes/pkg/client/*` | `k8s.io/client-go/*` |

### 2. Client-go 重组（BREAKING）
- `k8s.io/kubernetes/pkg/client/informers` → `k8s.io/client-go/informers`
- `k8s.io/kubernetes/pkg/client/listers` → `k8s.io/client-go/listers`
- `k8s.io/kubernetes/pkg/client-go/kubernetes` → `k8s.io/client-go/kubernetes`

### 3. 内部版本 API 移除
- `k8s.io/kubernetes/pkg/client/informers/informers_generated/internalversion` 已移除
- 必须使用外部版本 API (external versions)

### 4. 插件路径变更
- `k8s.io/kubernetes/plugin/pkg/auth/authorizer/rbac` → `k8s.io/kubernetes/staging/src/k8s.io/apiserver/pkgauthorization`

### 5. 命令行工具选项路径变更
- `k8s.io/kubernetes/cmd/kube-apiserver/app/options` → `k8s.io/kubernetes/staging/src/k8s.io/apiserver/pkg/server/options`
- `k8s.io/kubernetes/cmd/kube-proxy/app` → `k8s.io/kubernetes/staging/src/k8s.io/kube-proxy/cmd options`
- `k8s.io/kubernetes/cmd/kubelet/app/options` → `k8s.io/kubernetes/staging/src/k8s.io/kubelet/cmd/options`

## Impact
- **受影响的规格**: Kubernetes API 集成、Master 配置、Node 配置
- **受影响的代码**:
  - `pkg/cmd/server/kubernetes/master/master_config.go`
  - `pkg/cmd/server/kubernetes/node/node_config_test.go`
  - `pkg/cmd/server/origin/master_config.go`
  - 所有使用旧版 Kubernetes import 的文件

## ADDED Requirements

### Requirement: 新的 API Import 结构
系统必须支持从 `k8s.io/api`、`k8s.io/client-go`、`k8s.io/apiserver` 导入 Kubernetes 类型。

#### Scenario: 导入 core/v1 类型
- **WHEN** 代码需要使用 Pod 类型
- **THEN** 应从 `k8s.io/api/core/v1` 导入

### Requirement: Client-go Informers
系统必须使用 `k8s.io/client-go/informers` 替代旧的内部版本。

#### Scenario: 创建 SharedInformerFactory
- **WHEN** 需要创建共享的 Informer 工厂
- **THEN** 使用 `k8s.io/client-go/informers.NewSharedInformerFactory`

## MODIFIED Requirements

### Requirement: 内部版本 API 迁移
所有使用 `k8s.io/kubernetes/pkg/client/informers/informers_generated/internalversion` 的代码必须迁移到外部版本 API。

**旧代码**:
```go
kinternalinformers "k8s.io/kubernetes/pkg/client/informers/informers_generated/internalversion"
```

**新代码**:
```go
kinformers "k8s.io/client-go/informers"
```

## REMOVED Requirements

### Requirement: 废弃的内部 API 包
**Reason**: Kubernetes 1.20+ 移除了所有内部版本 API
**Migration**: 必须迁移到外部版本 API 或使用自动生成的客户端

### Requirement: 旧版 pkg/api 导入
**Reason**: `k8s.io/kubernetes/pkg/api` 在新版本中已移除
**Migration**: 使用 `k8s.io/api/core/v1` 替代
