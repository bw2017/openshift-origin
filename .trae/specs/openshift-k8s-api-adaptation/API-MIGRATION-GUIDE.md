# Kubernetes 1.35.3 API 迁移分析报告

## 概述
从 Kubernetes v1.11 升级到 v1.35.3 涉及**重大架构重组**。本文档详细分析需要修改的 API 路径。

## 仍可兼容的 API（无需修改）

以下 API 包在 Kubernetes 1.35.3 中仍保留原路径：

✅ `k8s.io/kubernetes/cmd/kube-apiserver/app/options` - ServerRunOptions
✅ `k8s.io/kubernetes/pkg/api/legacyscheme` - Scheme, Codecs, ParameterCodec
✅ `k8s.io/kubernetes/pkg/apis/core` - 核心 API 类型
✅ `k8s.io/kubernetes/pkg/apis/apps` - Apps API 类型
✅ `k8s.io/kubernetes/pkg/apis/autoscaling` - Autoscaling API
✅ `k8s.io/kubernetes/pkg/apis/batch` - Batch API
✅ `k8s.io/kubernetes/pkg/apis/extensions` - Extensions API
✅ `k8s.io/kubernetes/pkg/apis/networking` - Networking API
✅ `k8s.io/kubernetes/pkg/apis/policy` - Policy API
✅ `k8s.io/kubernetes/pkg/apis/storage` - Storage API
✅ `k8s.io/kubernetes/pkg/registry/core/endpoint` - Endpoint 注册表
✅ `k8s.io/kubernetes/pkg/registry/core/endpoint/storage` - Endpoint 存储
✅ `k8s.io/kubernetes/pkg/version` - 版本信息

## 需要重大重构的 API

### 1. master.Config → controlplaneapiserver.Config

**旧路径**: `k8s.io/kubernetes/pkg/master`
```go
// 旧代码
import "k8s.io/kubernetes/pkg/master"
config := &master.Config{
    GenericConfig: genericConfig,
    ExtraConfig: master.ExtraConfig{
        MasterCount: 1,
        // ...
    },
}
```

**新路径**: `k8s.io/kubernetes/pkg/controlplane/apiserver`
```go
// 新代码
import controlplaneapiserver "k8s.io/kubernetes/pkg/controlplane/apiserver"
config := &controlplane.Config{
    ControlPlane: controlplaneapiserver.Config{
        Generic: genericConfig,
        Extra: controlplaneapiserver.Extra{
            // ...
        },
    },
}
```

### 2. 新的 Extra 字段

**新增字段**:
- `ClusterAuthenticationInfo` - 集群认证信息
- `PeerProxy` - 对等代理
- `ServiceAccountIssuer` - ServiceAccount 颁发者
- `ExtendExpiration` - 扩展过期时间

**移除字段**:
- `APIServerServicePort` - 已移除
- `KubernetesServiceNodePort` - 移到 options
- `ClientCARegistrationHook` - 结构变更

### 3. API Resource Config Source

**旧代码**:
```go
master.DefaultAPIResourceConfigSource()
```

**新代码**:
```go
import serverstorage "k8s.io/apiserver/pkg/server/storage"
resourceConfig := serverstorage.NewResourceConfig()
```

## 适配策略

### 阶段 1: 简单 Import 替换
1. 替换 `k8s.io/kubernetes/pkg/master` → `k8s.io/kubernetes/pkg/controlplane/apiserver`
2. 更新 `master.Config` → `controlplane.Config`
3. 更新 `master.ExtraConfig` → `controlplane.Extra`

### 阶段 2: 结构适配
1. 更新所有 Config 字段引用
2. 适配新的 Extra 字段
3. 更新 API Resource Config Source 创建方式

### 阶段 3: 功能适配
1. 实现新的认证配置
2. 配置 ServiceAccount Issuer
3. 更新 Endpoint Reconciler

## 受影响的文件列表

1. `pkg/cmd/server/kubernetes/master/master_config.go` - 需要重大重构
2. `pkg/cmd/server/origin/master_config.go` - 需要更新 Config 类型引用
3. `pkg/cmd/server/kubernetes/node/node_config_test.go` - 需要检查 options 变更
