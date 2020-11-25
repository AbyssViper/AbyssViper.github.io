# Operator-sdk V1.2 简单案例快速入手




## 准备

### 案例简介

案例来自于 [《Programming-Kubernetes》](https://programming-kubernetes.info/)文章中的 cloud native at ，用于实现一个云原生的 at 命令, 

**效果：** 在yaml配置中 `schedule` 时间到达后，创建 busybox 镜像的 Pod，执行yaml配置中的 `command`

具体实现效果如下，通过以下格式的 yaml 进行定义

```shell
$ cat programming-kubernetes_v1alpha1_at.yaml
```

```yaml
apiVersion: programming-kubernetes.viper.run/v1alpha1
Kind: At
metadata:
  name: at-demo
spec:
  command: echo QAQ
  schedule: 2020-11-18
```

通过如下命令进行发布以及查看

```shell
$ kubectl apply -f programming-kubernetes_v1alpha1_at.yaml
$ kubectl get ats
```



### 环境介绍

| 组件         | 版本                 |
| ------------ | -------------------- |
| 开发环境     | Linux_x86_64         |
| kubernetes   | v1.16.6              |
| kubectl      | v1.16.6              |
| go           | go1.14.4 linux/amd64 |
| operator-sdk | v1.2.0               |
| IDE          | GoLand               |



## 初始化

### Go环境依赖

需要提前开启 go mod，配置  goproxy

```shell
export GO111MODULE=on
export GOPROXY=https://mirrors.aliyun.com/goproxy/
```

### 项目初始化

初始化项目，新版本的 operator-sdk 使用的 --domain进行指定域名，比如本次项目我们期望的是 programming-kubernetes.viper.run, 此处domain指定好 viper.run 之后，api创建的时候只需要指定 group 为 programming-kubernetes 即可

```shell
$ mkdir /workspace/programming-kubernetes
$ operator-sdk init --domain=viper.run --repo=github.com/AbyssViper/programming-kubernetes
```

![项目初始化](https://gitee.com/AbyssViper/pic/raw/master/images/operator-sdk-cnat/project-init.png "项目初始化")

### API创建

创建 API ，各个字段的含义 apiVersion: `group.domain/version`, kind 对应 yaml 配置中的 Kind， 这里我们设置 controller 为 true，同时按照 group 与 version 定义创建 controller。

```shell
$ operator-sdk create api --group programming-kubernetes --version v1alpha1 --kind At --resource=true --controller=true
```

![创建api](https://gitee.com/AbyssViper/pic/raw/master/images/operator-sdk-cnat/project-create-api.png "创建api")



## 发布测试

接下来进行一下发布测试，会创建对应的 CRD

```shell
$ make install
$ kubectl get crds | grep ats.programming-kubernetes.viper.run
```

![CRD发布测试](https://gitee.com/AbyssViper/pic/raw/master/images/operator-sdk-cnat/crds-created-test.png "CRD发布测试")



将自带的 sample yaml 发布至 Kubernetes，并且通过 kubectl get 查看发布情况

```shell
$ kubectl get ats
```

![发布并查看](https://gitee.com/AbyssViper/pic/raw/master/images/operator-sdk-cnat/crds-created-test-get-ats.png "发布并查看")

通过如下命令，对于已经发布的 crd 进行清理

```shell
make uninstall
```

![清理](https://gitee.com/AbyssViper/pic/raw/master/images/operator-sdk-cnat/crds-created-test-clean.png "清理")



## 编写业务逻辑

### 项目结构

一般情况下，对于业务逻辑，我们主要关注的是  `at_types.go` 与 `at_controller.go`,  yaml文件中对自定义资源使用的字段定义，是在 `at_types.go` 中定义的，相关的业务逻辑是需要在 `at_controller.go`中进行编写的。

![项目结构](https://gitee.com/AbyssViper/pic/raw/master/images/operator-sdk-cnat/project-struct.png "项目结构")



### 定义结构体

在 `at_types.go` 中进行 yaml 的映射结构定义

对于 spec 结构体，我们需要 `Schedule` `Command` 两个字段声明

```go
type AtSpec struct {
	// INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
	// Important: Run "make" to regenerate code after modifying this file

	// Schedule need UTC time format. Example: 2006-05-14T09:23:00Z
	Schedule string `json:"schedule,omitempty"`
	// Linux command will execute when be scheduled.
	Command string `json:"command,omitempty"`
}
```

而对于 status 结构体，我们需要一个 `Phase` 字段来存储当前 `At` 实例所处的阶段

```go
type AtStatus struct {
	// INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
	// Important: Run "make" to regenerate code after modifying this file

	// Phase set instance's status.
	Phase string `json:"phase,omitempty"`
}
```

既然有了 status 对应的状态，我们需要对状态常量进行个标识，所以 `at_types.go` 中还需要声明如下常量，可以看到对于我们的 `At` 实例有 `PENDING`, `RUNNING`, `DONE` 三个状态

```go
const (
	PhasePending = "PENDING"
	PhaseRunning = "RUNNING"
	PhaseDone    = "DONE"
)
```

相关的类型结构体映射已经定义好了，接下来就是实现相关的逻辑



### 实现逻辑

#### 逻辑结构

对于实现一个定时的at命令，基本实现逻辑分为以下几步：

- 

具体的实现逻辑在控制循环中进行控制，在controllers.at_controller.go 中的 Reconcile 函数中进行实现

```go
// +kubebuilder:rbac:groups=programming-kubernetes.viper.run,resources=ats,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=programming-kubernetes.viper.run,resources=ats/status,verbs=get;update;patch

func (r *AtReconciler) Reconcile(req ctrl.Request) (ctrl.Result, error) {
	_ = context.Background()
	_ = r.Log.WithValues("at", req.NamespacedName)

	// your logic here

	return ctrl.Result{}, nil
}
```





- 控制循环中判定当前 At 实例的状态
- 如果实例状态为 空，则代表新的实例，需要初始化添加 PENDING 状态
- 如果是 PENDING 状态，判定 schedule 字段跟当前时间比较，是否已经到达了需要执行 command 的时机，如果未到跳过本次循环。如果到达了需要执行的时机，更改状态为 RUNNING
- 如果状态为 RUNNING, 则以 busybox 镜像创建 Pod 实例，并且执行，如果 Pod 状态已经为 Success 或者是 Fail 状态，则更改实例状态为 Done
- 如果实例状态为 Done 则代表本次已经执行完成，返回



```go
// +kubebuilder:rbac:groups=programming-kubernetes.viper.run,resources=ats,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=programming-kubernetes.viper.run,resources=ats/status,verbs=get;update;patch

func (r *AtReconciler) Reconcile(req ctrl.Request) (ctrl.Result, error) {
	ctx := context.Background()
	reqLogger := r.Log.WithValues("at", req.NamespacedName)

	reqLogger.Info("----- Reconciling At -----")
	// Create At instance
	atInstance := &cnatv1alpha1.At{}
	// Try to get cloud native at instance.
	err := r.Get(ctx, req.NamespacedName, atInstance)
	if err != nil {
		// Request object not found.
		if errors.IsNotFound(err) {
			return ctrl.Result{}, nil
		}
		// Other error.
		return ctrl.Result{}, err
	}

	// If instance's phase not set (string type default ""), set PENDING default.
	if atInstance.Status.Phase == "" {
		atInstance.Status.Phase = cnatv1alpha1.PhasePending
	}

	// Switch instance phase, switch logic.
	switch atInstance.Status.Phase {
	case cnatv1alpha1.PhasePending:
		
	case cnatv1alpha1.PhaseRunning:
		
	case cnatv1alpha1.PhaseDone:
		reqLogger.Info("Phase:", "status", cnatv1alpha1.PhaseDone)
		return ctrl.Result{}, nil
	default:
		reqLogger.Info("Unknown instance status.")
		return ctrl.Result{}, nil
	}
	// Update this time reconcile status to the respective phase.
	if err := r.Status().Update(ctx, atInstance); err != nil {
		return ctrl.Result{}, err
	}
	return ctrl.Result{}, nil
}
```



```go
reqLogger.Info("Phase:", "status", cnatv1alpha1.PhasePending)
// Violate field from config yaml.
targetSchedule := atInstance.Spec.Schedule
// Violate schedule.
reqLogger.Info("Violate schedule format.", "schedule: ", targetSchedule)
timeNow := time.Now().UTC()
local, _ := time.LoadLocation("Asia/Shanghai")
layout := "2006-01-02T15:04:05Z"
s, err := time.ParseInLocation(layout, targetSchedule, local)
if err != nil {
    reqLogger.Error(err, "Phase schedule error.")
    // Requeue.
    return ctrl.Result{}, err
}
diffTime := s.Sub(timeNow)
reqLogger.Info("Schedule parse diff time end.", "result", diffTime)
// Not this time.
if diffTime > 0 {
    // Not this time, requeue after time diff.
    return ctrl.Result{RequeueAfter: diffTime}, nil
}
// Time now, will execute command.
reqLogger.Info("Ready to phase RUNNING.", "command", atInstance.Spec.Command)
// Change status, next requeue will execute case PhaseRUNNING.
atInstance.Status.Phase = cnatv1alpha1.PhaseRunning
```



```go
reqLogger.Info("Phase:", "status", cnatv1alpha1.PhaseRunning)
labels := map[string]string{
    "app": atInstance.Name,
}
exePod := &corev1.Pod{
    ObjectMeta: metav1.ObjectMeta{
        Name:      atInstance.Name + "-pod",
        Namespace: atInstance.Namespace,
        Labels:    labels,
    },
    Spec: corev1.PodSpec{
        Containers: []corev1.Container{
            {
                Name:    "busybox",
                Image:   "busybox",
                Command: strings.Split(atInstance.Spec.Command, " "),
            },
        },
        RestartPolicy: corev1.RestartPolicyOnFailure,
    },
}
if err := controllerutil.SetControllerReference(atInstance, exePod, r.Scheme); err != nil {
    // requeue with error.
    return ctrl.Result{}, err
}
getPod := &corev1.Pod{}
// Try to get exePod now.
if err := r.Get(ctx, types.NamespacedName{Name: exePod.Name, Namespace: exePod.Namespace}, getPod); err != nil {
    if errors.IsNotFound(err) {
        if err := r.Create(ctx, exePod); err != nil {
            return ctrl.Result{}, err
        }
        reqLogger.Info("Pod create success.", "name", exePod.Name)
    } else {
        return ctrl.Result{}, err
    }
} else if getPod.Status.Phase == corev1.PodFailed || getPod.Status.Phase == corev1.PodSucceeded {
    reqLogger.Info("container terminal", "reason", getPod.Status.Reason, "message", getPod.Status.Message)
    atInstance.Status.Phase = cnatv1alpha1.PhaseDone
} else {
    return ctrl.Result{}, nil
}
```










