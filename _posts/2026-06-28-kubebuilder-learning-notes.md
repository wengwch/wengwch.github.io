---
title: Kubebuilder 学习笔记：从 CRD 到多版本、多 Group、外部资源与多 Controller
date: 2026-06-28
categories:
  - Kubernetes
  - Kubebuilder
tags:
  - Kubernetes
  - Kubebuilder
  - CRD
  - Controller
  - Operator
---


在 Kubernetes 生态中，如果我们希望扩展 Kubernetes API，最常见的方式就是开发 **CRD + Controller/Operator**。Kubebuilder 是官方推荐的 Operator 开发框架之一，它基于 `controller-runtime`，可以帮助我们快速生成 CRD、Controller、Webhook、RBAC、部署清单等工程代码。

这篇文章整理我对 Kubebuilder 的一些核心学习内容，包括：

```text
1. CRD Subresources
2. API 类型如何被其他项目引用
3. 多版本 API
4. Submodule Layout
5. 多 Group
6. External Resources
7. Multiple Controllers per Resource
```

<!-- more -->

---

## 1. CRD Subresources

Kubebuilder 中最常见的 subresource 有两个：

```yaml
subresources:
  status: {}
  scale: {}
```

它们对应 Kubernetes API 的子路径：

```text
/apis/<group>/<version>/namespaces/<ns>/<resources>/<name>/status
/apis/<group>/<version>/namespaces/<ns>/<resources>/<name>/scale
```

### 1.1 status subresource

`status` 用来隔离期望状态和实际状态。

CRD 通常分为：

```yaml
spec:
  replicas: 3
  image: nginx:1.27

status:
  readyReplicas: 2
  phase: Running
```

语义上：

| 字段       | 谁写                    | 含义   |
| -------- | --------------------- | ---- |
| `spec`   | 用户 / 平台 / GitOps      | 期望状态 |
| `status` | controller / operator | 实际状态 |

开启 status subresource 后，普通更新主要修改 `spec`，controller 更新状态时应该走：

```go
client.Status().Update(ctx, obj)
```

而不是：

```go
client.Update(ctx, obj)
```

Kubebuilder 中开启方式：

```go
// +kubebuilder:subresource:status
type App struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   AppSpec   `json:"spec,omitempty"`
    Status AppStatus `json:"status,omitempty"`
}
```

然后重新生成 CRD：

```bash
make manifests
```

生成的 CRD 中会出现：

```yaml
subresources:
  status: {}
```

### 1.2 scale subresource

`scale` 用于让自定义资源接入 Kubernetes 标准扩缩容体系，比如：

```bash
kubectl scale app demo --replicas=5
```

或者让 HPA 作用在你的 CRD 上。

示例：

```yaml
subresources:
  status: {}
  scale:
    specReplicasPath: .spec.replicas
    statusReplicasPath: .status.replicas
    labelSelectorPath: .status.selector
```

Kubebuilder marker：

```go
// +kubebuilder:subresource:status
// +kubebuilder:subresource:scale:specpath=.spec.replicas,statuspath=.status.replicas,selectorpath=.status.selector
type App struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   AppSpec   `json:"spec,omitempty"`
    Status AppStatus `json:"status,omitempty"`
}
```

其中：

```go
type AppSpec struct {
    Replicas *int32 `json:"replicas,omitempty"`
}

type AppStatus struct {
    Replicas int32  `json:"replicas,omitempty"`
    Selector string `json:"selector,omitempty"`
}
```

一句话总结：

> `status` 用于隔离实际状态，`scale` 用于接入 Kubernetes 标准扩缩容接口。

---

## 2. Kubebuilder 的 API 类型如何引入到其他项目

Kubebuilder 生成的“类”本质是 Go struct，例如：

```go
api/v1/app_types.go
```

里面定义了：

```go
type App struct {}
type AppSpec struct {}
type AppStatus struct {}
```

其他 Go 项目如果想使用这些类型，通常直接 import 你的 `api/v1` 包。

假设 Kubebuilder 项目路径是：

```text
github.com/example/app-operator
```

其他项目中执行：

```bash
go get github.com/example/app-operator@v0.1.0
```

然后使用：

```go
import appv1 "github.com/example/app-operator/api/v1"
```

创建 client 时需要注册 scheme：

```go
scheme := runtime.NewScheme()

if err := appv1.AddToScheme(scheme); err != nil {
    panic(err)
}

c, err := client.New(cfg, client.Options{
    Scheme: scheme,
})
```

查询 CR：

```go
var app appv1.App

err := c.Get(ctx, client.ObjectKey{
    Namespace: "default",
    Name:      "demo",
}, &app)
```

创建 CR：

```go
obj := &appv1.App{
    ObjectMeta: metav1.ObjectMeta{
        Name:      "demo",
        Namespace: "default",
    },
    Spec: appv1.AppSpec{
        Image:    "nginx:1.27",
        Replicas: 3,
    },
}

err := c.Create(ctx, obj)
```

核心点是：

```go
appv1.AddToScheme(scheme)
```

因为 controller-runtime client 需要知道：

```text
Go struct <-> apiVersion/kind
```

的映射关系。

如果你的 CRD 类型会被多个项目复用，建议把 API types 拆成单独 module，避免其他项目被迫依赖整个 Operator 工程。

---

## 3. Kubebuilder 多版本 API

多版本 API 是指同一个 CRD 同时支持多个版本，比如：

```yaml
apiVersion: apps.example.com/v1alpha1
kind: App
```

和：

```yaml
apiVersion: apps.example.com/v1beta1
kind: App
```

它们是同一个资源的不同 API 版本。

典型目录结构：

```text
api/
  v1alpha1/
    app_types.go
    groupversion_info.go
    zz_generated.deepcopy.go

  v1beta1/
    app_types.go
    groupversion_info.go
    app_conversion.go
    zz_generated.deepcopy.go
```

### 3.1 storage version

CRD 可以暴露多个版本，但 etcd 中通常只有一个主存储版本。

Kubebuilder 中用 marker 指定：

```go
// +kubebuilder:storageversion
type App struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   AppSpec   `json:"spec,omitempty"`
    Status AppStatus `json:"status,omitempty"`
}
```

生成 CRD 后类似：

```yaml
spec:
  versions:
    - name: v1alpha1
      served: true
      storage: false
    - name: v1beta1
      served: true
      storage: true
```

注意：

> 一个 CRD 只能有一个 `storage: true` 版本。

### 3.2 conversion webhook

如果用户提交的是：

```yaml
apiVersion: apps.example.com/v1alpha1
kind: App
```

但是 storage version 是 `v1beta1`，API Server 需要把 `v1alpha1` 转成 `v1beta1` 再存储。

这时就需要 conversion webhook。

Kubebuilder 推荐使用 Hub-Spoke 模型：

```text
Hub:   v1beta1
Spoke: v1alpha1
```

Hub 版本：

```go
// api/v1beta1/app_conversion.go
package v1beta1

func (*App) Hub() {}
```

Spoke 版本：

```go
func (src *App) ConvertTo(dstRaw conversion.Hub) error
func (dst *App) ConvertFrom(srcRaw conversion.Hub) error
```

示例：

旧版本：

```go
// v1alpha1
type AppSpec struct {
    Image string `json:"image,omitempty"`
    Size  int32  `json:"size,omitempty"`
}
```

新版本：

```go
// v1beta1
type AppSpec struct {
    Image    string `json:"image,omitempty"`
    Replicas *int32 `json:"replicas,omitempty"`
}
```

转换代码：

```go
func (src *App) ConvertTo(dstRaw conversion.Hub) error {
    dst := dstRaw.(*appsv1beta1.App)

    dst.ObjectMeta = src.ObjectMeta
    dst.Spec.Image = src.Spec.Image

    replicas := src.Spec.Size
    dst.Spec.Replicas = &replicas

    dst.Status.Phase = src.Status.Phase

    return nil
}

func (dst *App) ConvertFrom(srcRaw conversion.Hub) error {
    src := srcRaw.(*appsv1beta1.App)

    dst.ObjectMeta = src.ObjectMeta
    dst.Spec.Image = src.Spec.Image

    if src.Spec.Replicas != nil {
        dst.Spec.Size = *src.Spec.Replicas
    }

    dst.Status.Phase = src.Status.Phase

    return nil
}
```

### 3.3 多版本 API 的注意点

多版本转换最容易出现的问题是字段丢失。

比如新版本有字段：

```go
EnableMetrics bool `json:"enableMetrics,omitempty"`
```

旧版本没有这个字段。那么：

```text
v1beta1 -> v1alpha1 -> v1beta1
```

可能导致 `enableMetrics` 丢失。

解决方式包括：

```text
1. 不在旧版本暴露无法表达的新能力
2. 用 annotation 保存无法表达的字段
3. 旧版本只保留兼容查询，不推荐作为编辑入口
```

Controller 建议只面向 Hub/storage version 写业务逻辑：

```go
var app appsv1beta1.App

err := r.Get(ctx, req.NamespacedName, &app)
```

不要在一个 Reconciler 里同时处理多个版本的业务模型。

---

## 4. Kubebuilder Submodule Layout

Kubebuilder 的 submodule 通常不是 Git submodule，而是 Go 子模块布局。

常见目录：

```text
app-operator/
  go.work

  api/
    go.mod
    v1/
      app_types.go
      groupversion_info.go
      zz_generated.deepcopy.go

  operator/
    go.mod
    cmd/main.go
    internal/controller/
    config/
```

API module：

```text
github.com/example/app-operator/api
```

Operator module：

```text
github.com/example/app-operator/operator
```

### 4.1 为什么要拆 submodule

适合这些场景：

```text
1. 多个项目要 import 你的 CRD Go 类型
2. 其他 controller 要 watch 你的 CRD
3. 平台服务要用强类型 client 创建/查询你的 CRD
4. 不想让使用方依赖整个 operator 工程
5. 希望 API types 单独发布版本
```

API module 只放类型定义：

```text
api/
  go.mod
  v1/
    groupversion_info.go
    app_types.go
    zz_generated.deepcopy.go
```

Operator module 负责运行 controller：

```text
operator/
  go.mod
  cmd/main.go
  internal/controller/
  config/
```

### 4.2 operator 引用 api module

`operator/go.mod`：

```go
module github.com/example/app-operator/operator

go 1.22

require (
    github.com/example/app-operator/api v0.0.0
)

replace github.com/example/app-operator/api => ../api
```

`operator/cmd/main.go`：

```go
import appv1 "github.com/example/app-operator/api/v1"

func init() {
    utilruntime.Must(clientgoscheme.AddToScheme(scheme))
    utilruntime.Must(appv1.AddToScheme(scheme))
}
```

本地可以用 `go.work`：

```bash
go work init ./api ./operator
```

生成：

```go
go 1.22

use (
    ./api
    ./operator
)
```

### 4.3 什么时候不建议拆

小项目不建议过早拆 submodule。

因为拆分之后会带来额外成本：

```text
1. go.mod 版本管理
2. go.work 本地开发
3. Makefile 路径调整
4. PROJECT 文件调整
5. controller-gen paths 调整
6. tag 发布规则调整
```

如果只是一个普通 Operator，保持单 module 更简单。

---

## 5. Kubebuilder 多 Group

多 group 是指一个 Operator 项目中管理多个 API Group，例如：

```text
apps.example.com/v1
batch.example.com/v1
infra.example.com/v1
```

默认 Kubebuilder 是 single-group layout：

```text
api/
  v1/
    app_types.go

internal/controller/
  app_controller.go
```

开启 multi-group 后：

```text
api/
  apps/
    v1/
      app_types.go

  batch/
    v1/
      task_types.go

internal/controller/
  apps/
    app_controller.go

  batch/
    task_controller.go
```

### 5.1 开启 multi-group

执行：

```bash
kubebuilder edit --multigroup=true
```

然后创建 API：

```bash
kubebuilder create api \
  --group apps \
  --version v1 \
  --kind App
```

再创建另一个 group：

```bash
kubebuilder create api \
  --group batch \
  --version v1 \
  --kind Task
```

生成目录：

```text
api/
  apps/
    v1/
      app_types.go
      groupversion_info.go

  batch/
    v1/
      task_types.go
      groupversion_info.go
```

### 5.2 main.go 注册多个 group

因为多个 package 都叫 `v1`，import 时必须起别名：

```go
import (
    appsv1 "example.com/project/api/apps/v1"
    batchv1 "example.com/project/api/batch/v1"
)

func init() {
    utilruntime.Must(clientgoscheme.AddToScheme(scheme))

    utilruntime.Must(appsv1.AddToScheme(scheme))
    utilruntime.Must(batchv1.AddToScheme(scheme))
}
```

### 5.3 多 group 和多 version 可以组合

例如：

```text
api/
  apps/
    v1alpha1/
    v1beta1/
    v1/

  batch/
    v1alpha1/
    v1/
```

对应：

```yaml
apiVersion: apps.example.com/v1
kind: App
```

和：

```yaml
apiVersion: batch.example.com/v1
kind: Task
```

注意：

```text
apps.example.com/App
batch.example.com/App
```

即使 Kind 名字一样，也属于完全不同的资源。

### 5.4 什么时候使用多 group

适合：

```text
同一个 operator 管理多个领域模型
不同资源生命周期、权限边界明显
API 领域需要清晰分层
```

例如：

```text
apps.example.com
  App
  Component
  Release

infra.example.com
  Cluster
  NodePool
  Network

ops.example.com
  Backup
  Restore
  Schedule
```

如果只是：

```text
App
AppConfig
AppBackup
AppMonitor
```

它们属于同一个业务域，通常一个 group 就够：

```text
apps.example.com
```

一句话总结：

> 多 group 解决的是“一个 Operator 管理多个 API 领域”的结构问题，不是多版本，也不是 submodule。

---

## 6. Using External Resources

Kubebuilder 里的 external resources 容易有两种含义。

### 6.1 使用外部 API 类型

比如另一个 Operator 已经定义了 CRD：

```yaml
apiVersion: infra.example.com/v1
kind: Database
```

Go 类型在：

```text
github.com/example/database-operator/api/v1
```

当前项目不想重新定义这个 CRD，只想给它写 controller。

可以使用：

```bash
kubebuilder create api \
  --group infra \
  --version v1 \
  --kind Database \
  --controller \
  --resource=false \
  --external-api-path=github.com/example/database-operator/api/v1 \
  --external-api-domain=example.com
```

关键参数：

| 参数                      | 作用                 |
| ----------------------- | ------------------ |
| `--controller`          | 生成 controller      |
| `--resource=false`      | 不生成本项目 API 类型和 CRD |
| `--external-api-path`   | 外部 API Go 包路径      |
| `--external-api-domain` | 外部 API 的 domain    |

然后在 `main.go` 中注册外部类型：

```go
import databasev1 "github.com/example/database-operator/api/v1"

func init() {
    utilruntime.Must(clientgoscheme.AddToScheme(scheme))
    utilruntime.Must(databasev1.AddToScheme(scheme))
}
```

Controller：

```go
func (r *DatabaseReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&databasev1.Database{}).
        Complete(r)
}
```

这种场景下，你的主资源就是外部 CRD。

### 6.2 监听外部管理的资源

另一种场景是：你的 CRD 引用了外部资源。

比如：

```yaml
apiVersion: apps.example.com/v1
kind: App
metadata:
  name: demo
spec:
  configMapName: app-config
```

这里 `ConfigMap/app-config` 不是你的 controller 创建的，不应该用 `Owns()`。

应该用：

```go
Watches(
    &corev1.ConfigMap{},
    handler.EnqueueRequestsFromMapFunc(r.findAppsForConfigMap),
)
```

核心思想：

```text
ConfigMap 变化
    ↓
MapFunc 找到引用它的 App
    ↓
把这些 App 加入 reconcile queue
```

推荐使用 FieldIndexer：

```go
if err := mgr.GetFieldIndexer().IndexField(
    context.Background(),
    &appv1.App{},
    ".spec.configMapName",
    func(rawObj client.Object) []string {
        app := rawObj.(*appv1.App)
        if app.Spec.ConfigMapName == "" {
            return nil
        }
        return []string{app.Spec.ConfigMapName}
    },
); err != nil {
    return err
}
```

MapFunc：

```go
func (r *AppReconciler) findAppsForConfigMap(
    ctx context.Context,
    obj client.Object,
) []reconcile.Request {
    cm := obj.(*corev1.ConfigMap)

    var appList appv1.AppList
    if err := r.List(ctx, &appList,
        client.InNamespace(cm.Namespace),
        client.MatchingFields{
            ".spec.configMapName": cm.Name,
        },
    ); err != nil {
        return nil
    }

    requests := make([]reconcile.Request, 0, len(appList.Items))
    for _, app := range appList.Items {
        requests = append(requests, reconcile.Request{
            NamespacedName: types.NamespacedName{
                Namespace: app.Namespace,
                Name:      app.Name,
            },
        })
    }

    return requests
}
```

Setup：

```go
func (r *AppReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&appv1.App{}).
        Watches(
            &corev1.ConfigMap{},
            handler.EnqueueRequestsFromMapFunc(r.findAppsForConfigMap),
        ).
        Complete(r)
}
```

### 6.3 Owns 和 Watches 的区别

| 方法          | 用途                            | 示例                      |
| ----------- | ----------------------------- | ----------------------- |
| `For()`     | 主资源                           | 自己的 CRD                 |
| `Owns()`    | 当前 controller 创建并 owner 管理的资源 | Deployment、Service      |
| `Watches()` | 不拥有但需要监听的外部资源                 | ConfigMap、Secret、其他 CRD |

一句话总结：

> 外部资源如果不是你拥有的，不要用 `Owns()`，而是用 `Watches()` + `MapFunc` 把外部资源事件映射回主资源。

---

## 7. Multiple Controllers per Resource

Multiple controllers per resource 是指：同一个 Kind 可以有多个 named controller。

例如同一个 CRD：

```yaml
apiVersion: crew.example.com/v1
kind: Captain
```

同时有多个 controller：

```text
CaptainReconciler
CaptainBackupReconciler
CaptainMetricsReconciler
```

它们都 watch `Captain`，但负责不同逻辑。

### 7.1 使用场景

```text
CaptainReconciler
  - 管理 Deployment
  - 管理 Service
  - 回写 workload 状态

CaptainBackupReconciler
  - 管理 Backup CronJob
  - 管理备份 Secret
  - 回写 backup 状态

CaptainMetricsReconciler
  - 管理 ServiceMonitor
  - 回写 metrics 状态
```

### 7.2 PROJECT 配置

多 controller 格式：

```yaml
resources:
- api:
    crdVersion: v1
  controllers:
  - name: captain
  - name: captain-backup
  group: crew
  kind: Captain
  version: v1
```

如果要给已有资源新增一个 controller：

```bash
kubebuilder create api \
  --group crew \
  --version v1 \
  --kind Captain \
  --controller \
  --resource=false \
  --controller-name captain-backup
```

注意：

```bash
--resource=false
```

因为 API 类型已经存在，不要重复生成。

### 7.3 代码示例

主 controller：

```go
type CaptainReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

func (r *CaptainReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&crewv1.Captain{}).
        Owns(&appsv1.Deployment{}).
        Named("captain").
        Complete(r)
}
```

备份 controller：

```go
type CaptainBackupReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

func (r *CaptainBackupReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&crewv1.Captain{}).
        Owns(&batchv1.CronJob{}).
        Named("captain-backup").
        Complete(r)
}
```

`cmd/main.go` 中都要注册：

```go
if err = (&controller.CaptainReconciler{
    Client: mgr.GetClient(),
    Scheme: mgr.GetScheme(),
}).SetupWithManager(mgr); err != nil {
    os.Exit(1)
}

if err = (&controller.CaptainBackupReconciler{
    Client: mgr.GetClient(),
    Scheme: mgr.GetScheme(),
}).SetupWithManager(mgr); err != nil {
    os.Exit(1)
}
```

### 7.4 最大风险：写入冲突

多个 controller 管理同一个 resource 时，最容易出现：

```text
1. 两个 controller 都更新 status
2. 两个 controller 都添加/删除 finalizer
3. 两个 controller 都 patch 同一个 annotation
4. 两个 controller 都创建同名子资源
5. 两个 controller 都修改同一个 Deployment
```

所以一定要做职责隔离。

### 7.5 status 字段隔离

不推荐多个 controller 都写：

```go
captain.Status.Phase = "Ready"
```

推荐拆分 status：

```go
type CaptainStatus struct {
    Workload WorkloadStatus `json:"workload,omitempty"`
    Backup   BackupStatus   `json:"backup,omitempty"`
    Metrics  MetricsStatus  `json:"metrics,omitempty"`
}
```

约定：

```text
CaptainReconciler 只写 status.workload
CaptainBackupReconciler 只写 status.backup
CaptainMetricsReconciler 只写 status.metrics
```

更新 status 时尽量用 patch：

```go
base := captain.DeepCopy()

captain.Status.Backup.Phase = "Ready"

if err := r.Status().Patch(ctx, &captain, client.MergeFrom(base)); err != nil {
    return ctrl.Result{}, err
}
```

### 7.6 finalizer 隔离

不要多个 controller 共用一个 finalizer。

推荐：

```text
captain.crew.example.com/finalizer
captain-backup.crew.example.com/finalizer
captain-metrics.crew.example.com/finalizer
```

备份 controller 只处理自己的 finalizer：

```go
const backupFinalizer = "captain-backup.crew.example.com/finalizer"
```

### 7.7 子资源隔离

不同 controller 管理不同资源：

```text
CaptainReconciler 创建：
  deployment/demo
  service/demo

CaptainBackupReconciler 创建：
  cronjob/demo-backup
  secret/demo-backup-config

CaptainMetricsReconciler 创建：
  servicemonitor/demo
```

不要多个 controller 同时管理同一个对象，除非你明确使用了 Server-Side Apply field ownership 或非常严格的 patch 策略。

一句话总结：

> 同一个 Kind 可以有多个 controller，但必须隔离 status、finalizer、子资源和字段所有权，否则很容易产生资源竞争。

---

## 8. Kubebuilder 学习路线总结

如果从零学习 Kubebuilder，我建议按下面顺序掌握。

### 第一阶段：基础 Operator

先掌握：

```text
1. kubebuilder init
2. kubebuilder create api
3. CRD 类型定义
4. Reconcile 主循环
5. client.Get / Create / Update / Patch
6. RBAC marker
7. make manifests / make install / make deploy
```

理解核心模型：

```text
用户提交 CR
    ↓
API Server 存储 CR
    ↓
controller watch CR
    ↓
Reconcile 计算期望状态
    ↓
创建/更新 Deployment、Service 等资源
    ↓
回写 status
```

### 第二阶段：status 和资源归属

掌握：

```text
1. status subresource
2. finalizer
3. ownerReference
4. Owns()
5. controllerutil.SetControllerReference()
6. status condition
```

这是写稳定 Operator 的基础。

### 第三阶段：外部资源

掌握：

```text
1. Watches()
2. EnqueueRequestsFromMapFunc
3. FieldIndexer
4. 监听 ConfigMap / Secret / 外部 CRD
5. 处理资源缺失
6. 跨 namespace 引用安全
```

典型模式：

```text
External Resource 变化
    ↓
MapFunc 找到引用它的主 CR
    ↓
触发主 CR Reconcile
```

### 第四阶段：API 设计

掌握：

```text
1. CRD 字段设计
2. spec/status 分离
3. validation marker
4. defaulting webhook
5. validating webhook
6. conversion webhook
7. 多版本 API
```

API 一旦发布，后续兼容成本很高，所以字段设计要谨慎。

### 第五阶段：工程组织

掌握：

```text
1. 多 Group
2. 多版本
3. Submodule Layout
4. API module 独立发布
5. 多 Controller
6. controller 职责拆分
```

大型 Operator 项目一定会遇到这些问题。

---

## 9. 最佳实践总结

### 9.1 CRD 设计

```text
1. spec 表达期望状态
2. status 表达实际状态
3. controller 不要修改用户的 spec
4. status 更新使用 Status().Update 或 Status().Patch
5. 开启 status subresource
6. 需要 HPA/kubectl scale 时再开启 scale subresource
```

### 9.2 Controller 设计

```text
1. Reconcile 要幂等
2. 不要依赖事件内容，始终重新读取当前集群状态
3. 不要无限重试用户配置错误
4. 缺失依赖资源时写 status condition
5. 用 patch 减少冲突
6. 用 finalizer 处理外部资源清理
```

### 9.3 资源监听

```text
1. 主资源用 For()
2. 自己创建并 owner 管理的资源用 Owns()
3. 外部资源用 Watches() + MapFunc
4. 大量资源引用关系用 FieldIndexer
```

### 9.4 多版本 API

```text
1. 只设置一个 storage version
2. 使用 Hub-Spoke 模型做 conversion
3. controller 面向 storage/hub 版本写业务逻辑
4. 避免转换过程中丢字段
5. 稳定版本不要轻易做破坏性变更
```

### 9.5 多 Controller

```text
1. 不要为了拆而拆
2. 职责独立时才拆多个 controller
3. 每个 controller 使用不同 finalizer
4. 每个 controller 写不同 status 子字段
5. 每个 controller 管理不同 owned resources
6. 避免多个 controller 更新同一个对象的同一个字段
```

---

## 10. 总结

Kubebuilder 的核心不是生成代码，而是帮助我们按照 Kubernetes 的控制器模型开发扩展 API。

可以用一句话概括：

> Kubebuilder = CRD 类型定义 + controller-runtime Reconcile 模型 + RBAC/CRD/Webhook 生成工具链。

在简单场景下，一个 Kubebuilder 项目可能只需要：

```text
api/v1
internal/controller
```

但随着项目复杂度提升，就会逐步遇到：

```text
多版本 API
多 Group
API types 被其他项目复用
external resources
multiple controllers per resource
conversion webhook
submodule layout
```

这些能力并不是一开始都要用，而是在项目规模、API 生命周期、团队协作和复用需求上来之后才逐步引入。

我的实践建议是：

```text
小项目：
  单 group、单 version、单 module、单 controller

中型项目：
  单 group、多 Kind、必要时多 version

大型项目：
  多 group、多 version、API submodule、多个 controller 职责拆分

平台型项目：
  强类型 API module + dynamic client + external resources watch + 完整权限控制
```

最终目标不是把 Kubebuilder 的所有功能都用上，而是让 Operator 的 API 设计清晰、Reconcile 逻辑幂等、权限最小化、资源关系可追踪、版本演进可维护。
