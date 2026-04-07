# Kubernetes 1.35 内部版本 API 移除问题

## 问题描述

Kubernetes 1.20+ 移除了所有内部版本 API（internal versions）。OpenShift Origin 代码中大量使用了这些已移除的 API，导致编译失败。

## 受影响的代码

### 1. InternalKubeInformers 类型
```go
// 旧代码 (已移除)
kinternalinformers "k8s.io/kubernetes/pkg/client/informers/informers_generated/internalversion"
InternalKubeInformers kinternalinformers.SharedInformerFactory

// 使用方式
informers.Core().InternalVersion().Namespaces()
informers.Core().InternalVersion().LimitRanges()
informers.Core().InternalVersion().Services()
informers.Core().InternalVersion().Endpoints()
```

### 2. OpenShift 内部 Informers
```go
// 旧代码 (已移除)
authorizationinformer "github.com/openshift/origin/pkg/authorization/generated/informers/internalversion"
SecurityInformers      securityinformer.SharedInformerFactory
AuthorizationInformers authorizationinformer.SharedInformerFactory
QuotaInformers         quotainformer.SharedInformerFactory
```

## 迁移方案

### 方案 1: 使用外部版本 API（推荐）
将所有 `.InternalVersion()` 调用改为 `.V1()` 或 `.V1beta1()` 等外部版本。

```go
// 旧代码
informers.Core().InternalVersion().Namespaces()

// 新代码
informers.Core().V1().Namespaces()
```

### 方案 2: 生成新的外部版本 Informers
使用 `k8s.io/client-go/informers` 替代旧版内部版本：

```go
// 新代码
import kinformers "k8s.io/client-go/informers"
kubeInformers kinformers.SharedInformerFactory
```

## 需要修改的文件

1. `pkg/cmd/server/origin/master_config.go`
   - [ ] 移除 `kinternalinformers` import
   - [ ] 将 `InternalKubeInformers` 类型改为 `kinformers.SharedInformerFactory`
   - [ ] 将所有 `.InternalVersion()` 调用改为 `.V1()`

2. `pkg/cmd/server/origin/master.go`
   - [ ] 更新 `InternalKubeInformers` 引用

3. `pkg/cmd/server/origin/dns_server.go`
   - [ ] 将 `.InternalVersion()` 改为 `.V1()`

## 工作量评估

这是一个**重大的架构变更**，需要：
- 修改至少 3 个文件
- 替换所有 `.InternalVersion()` 调用
- 可能需要重新生成 OpenShift 的 client-go 代码

建议：
1. 使用外部版本 API 进行渐进式迁移
2. 考虑使用代码自动化工具进行批量替换
3. 进行充分的测试以确保功能正常
