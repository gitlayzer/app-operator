## CRD与Operator

### 1：引言

```shell
Costom Resource Define简称CRD，是Kubernetes（v1.7+）为提高扩展性，让开发者去自定义资源的一种方式，CRD资源可以动态注册到集群中，注册完毕后，用户可以通过kubectl来创建访问这个自定义的资源对象，类似于操作Pod一样，不过需要注意的是，CRD仅仅是资源的定义而已，还需要一个Controller去监听CRD的各种事件来添加自定义的业务逻辑。
```

### 2：CRD

```shell
如果说只是对CRD资源本身进行CRUD操作的话，不需要Controller也是可以实现的，相当于是只有数据存入了etcd中，而没有对这个数据的相关操作而已，比如我们可以定义一个CRD如下
```

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # name字段必须匹配下面的spec字段 <plural>.<group>
  name: crontabs.stable.example.com
spec:
  # group名用于REST API中定义：/apis/<group>/<version>
  group: stable.example.com
  # 列出自定义资源的所有API版本
  versions:
    # name：指定的是版本名称，比如v1，v1alpha1等
  - name: v1alpha1
    # 是否开启通过REST APIs访问 '/apis/<group>/<version>/...'
    served: true
    # 必须将一个且只有一个版本标记为存储版本
    storage: true
    # 定义自定义对象的声明规范
    schema:
      openAPIV3Schema:
        description: Define CronTab YAML Spec
        type: object
        properties:
          spec:
            type: object
            properties:
              cronSpec:
                type: string
              image:
                type: string
              replicas:
                type: integer
  # 定义作用范围： Namespaced（命名空间级别）或者Cluster（整个集群）
  scope: Namespaced
  names:
    # kind: 是sigular的一个驼峰形式定义，在资源清单中会使用
    kind: CronTab
    # plural：名字用于REST API中定义：/apis/<group>/<version>/<plural>
    plural: crontabs
    # singular: 名称用于CLI操作时的一个别名
    singular: crontab
    # shortNames: 相当于缩写形式
    shortNames:
    - ct
```

```shell
# 创建资源
[root@k-m-1 crds]# kubectl apply -f crds.yaml 
customresourcedefinition.apiextensions.k8s.io/crontabs.stable.example.com created
# 查看资源
[root@k-m-1 crds]# kubectl get customresourcedefinitions.apiextensions.k8s.io | grep example
crontabs.stable.example.com                           2023-06-13T01:11:58Z
# 查看存在的API
[root@k-m-1 crds]# kubectl api-resources | grep example
crontabs         ct                 stable.example.com/v1alpha1                 true              CronTab

# 需要注意的是v1.16版本以后已经是GA了，使用的是v1版本，之前都是v1beta1，定义规范部分有变化，所以需要注意版本变化。
# 这个地方和我们定义的普通资源对象比较类似，我们说我们可以随意定义一个自定义资源对象，但是在创建资源的时候，肯定不是任由我们随意去编写YAML的，当我们把上面的CRD文件提交给kubernetes之后，kubernetes会对我们的提交的声明文件进行校验，从定义可以看出CRD是基于OpenAPI V3 Schema 进行规范的，当然这种校验只是对于字段类型进行校验，比较初级，如果想要复杂的校验，这个时候需要通过kubernetes的admssion webhook来实现，关于校验的更多用法，可以去官方文档查看。

# 这个时候一个新的namespace级别的RESTful API就会被创建
/apis/stable/example.com/v1alpha1/namespace/*/contabs/...

# 然后我们可以使用这个API端点来创建和管理自定义的对象，这些对象的类型就是上面创建的CRD对象规范中的CronTab
```

```yaml
apiVersion: stable.example.com/v1alpha1
kind: CronTab
metadata:
  name: crontab-crd
spec:
  cronSpec: "* * * * */1"
  image: "busybox:latest"
  replicas: 1
```

```shell
# 创建资源
[root@k-m-1 crds]# kubectl apply -f crd-demo.yaml 
crontab.stable.example.com/crontab-crd created
# 查看资源
[root@k-m-1 crds]# kubectl get crontabs.stable.example.com 
NAME          AGE
crontab-crd   21s
# 使用简写查看资源
[root@k-m-1 crds]# kubectl get ct
NAME          AGE
crontab-crd   5s

# 到此为止这就是CRD的一个概念，它只是定义数据，但是至于怎么处理数据，需要下面的Controller来实现。
```

### 3：Controller

```shell
如上所说，现在CRD我们定义和创建完成了，但是也只是单纯的把资源清单数据存入了etcd中而已，并没有其他用处，因为我们没有去定义一个Controller来处理它。

官方提供了一个自定义Controller的示例：https://github.com/kubernetes/sample-controller，实现了：
1：如何注册资源Foo
2：如何创建，删除和查询Foo对象
3：如何监听Foo资源对象的变化情况

要想了解Controller的实现原理和方式，我们就需要了解下Client-go这个库的实现，Kubernetes部分代码也是基于这个库实现的，也包含了开发自定义控制器时可以使用的各种机制，这些机制在client-go源码的tools/cache目录下面有定义，

下图是Client-go中的各个组件时如何工作的以及我们要编写的自定义控制器代码的交互入口
```

![client-go](https://img-blog.csdnimg.cn/img_convert/ff6437d0f75e329e533f4cc5b549b90f.webp?x-oss-process=image/format,png)

```shell
示例地址：https://github.com/kubernetes/sample-controller
```

```shell
# client-go组件
1：Reflector：通过kubernetes API监控Kubernetes资源类型，采用List/Watch机制，可以Watch任何资源包括CRD添加object对象到FIFO队列，然后Informer会从队列取数据
2：Infomer：controller机制的基础，循环处理object对象从Reflector取出数据，然后将数据给到Indexer去缓存，提供对象事件的handler接口，只给Informer添加ResourceEventHandler实例的回调函数，去实现Onadd(obj interface{})，OnUpdate(obj interface{})和OnDelete(obj interface{})这三个方法，就可以处理好资源的创建，更新和删除操作了
3：Indexer：提供object对象的索引，是线程安全额，缓存对象信息

# controller组件
1：Informer reference：controller需要创建合适的Informer才能通过Informer reference操作资源对象
2：Indexer reference：controller创建Indexer reference然后去利用索引做相关的处理
3：Resource Event Handlers：Informer会调用这些Handlers
4：Work queue：Resource Event Handlers被调用后将key写到工作队列，这里的key相当于事件通知，然后根据取出事件后，做后续处理
5：Process Item：从工作队列取出key后进行后续处理，具体处理可以通过Indexer reference controller可以直接创建上述两个引用对象去处理，也可以采用工厂模式，官方都有案例。

client-go/tools/cache和定义controller的控制流
```

![informer](https://oscimg.oschina.net/oscnet/up-7a44aea37922ef7becad70872a768f46171.png)

```shell
如图主要的两个部分，一个发生在SharedIndexInformer中，另外一个是在自定义的控制器中。

1：Reflector：通过Kubernetes APIServer执行对象的ListAndWatch查询，记录和对象相关的三种事件类型Added，Updated，Deleted然后将它们传递到DeltaFIFO中去。
2：DeltaFIFO接收到事件和watch事件对应的对象后，然后将它们转换为Delta对象，这些Delta对象被附加到队列中去等待处理，对于已经删除的，会检查线程安全的store中是否已经存在该文件，从而避免在不存在某些内容的排队执行删除操作
3：Cache控制器（不要和自定义控制器混淆）：调用Pop()方法从DeltaFIFO队列中出队列，Delta对象将传递到SharedIndexInformer的HandlerDelta()方法中以进行下一步处理。
4：根据Delta对象的操作（事件）类型，首先在HandlerDeltas方法中通过indexer的方法将对对象保存到线程安全的Store中，然后通过SharedIndexInformer中的sharedProcessor的distribution()方法将这些对象发送到事件Handlers，这些事件处理器由自定义控制器通过ShareedInformer的方法比如AddEventHandlerWithResyncPeriod()进行注册
5：已注册的事件处理通过添加或更新事件的MetaNamespaceKeyFunc()或删除事件的DeletionHandingMetaNamespaceKeyFunc()将对象转换格式为namespace/name或只是name的key，然后将这个key添加到自定义控制器的workqueue中，workqueue的实现可以在util/workqueue中找到
6：自定义的控制器通过调用定义的handlers处理器从workqueue中pop一个key出来进行处理，handlers将调用indexer的GetByKey()从线程安全的store中获取对象，我们的业务逻辑就是在这个handlers里面实现

client-go中也有自定义Controller的样例代码，位于：k8s.io/client-go/examples/workqueue/main.go
```

### 4：Operator

```shell
Operator就可以看成是CRD和Controller的一种结合机制，Operator是一种思想，它集合了特定领域只是并通过CRD机制扩展了Kubernetes API资源，使用户管理Kubernetes的内置资源(Pod，Deployment等)一样创建，配置和应用管理程序，Operator是一个特定的应用程序的控制器，通过扩展Kubernetes API资源以代表Kubernetes用户创建，配置和管理复杂应用程序的实例，通过包含资源模型定义和控制器，通过Operator通常是为了实现某种特定软件(通常是有状态服务)的自动化运维。

我们完全可以通过上面的方式编写一个CRD对象，然后去手动实现一个对应的Controller就可以实现一个Operator，但是我们也发现从头开始去创建一个CRD控制器并不容易，需要对kubernetes的API有深入了解，并且RBAC集成，构建镜像，持续集成和部署等都需要很大工作量，为了解决这个问题，社区就退出了简单易用的Operator框架，比较主流的是kubebuilder和Operator Framework，这两个框架基本上差别不大，我们可以根据自己的习惯选择一个就可以了。
```

### 5：Operator Framework

```shell
Operator Framework是CoreOS开源的一个用于快速开发Operator的工具包，该框架包含两个主要部分：
1：Operator SDK：无需了解负载的Kubernetes API特性，即可让你根据自己的专业知识构建了一个Operator应用。
2：Operator Lifecycle Manager（0LM）：帮助你安装，更新和管理跨集群的运行中所有Operator(以及他们的相关服务)
```

![operator-sdk](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/operator-sdk-lifecycle.png)

```shell
Operator SDK提供以下几种工作流来开发一个新的Operator
1：使用SDK创建一个新的Operator项目
2：通过添加自定义资源（CRD）定义新的资源API
3：指定用SDK API来watch的资源
4：定义Operator的协调（reconcile）逻辑
5：使用Operator SDK构建并生成Operator部署清单文件

# 示例
我们平时在部署一个简单的WebServer到Kubernetes集群中的时候，都需要先编写一个Deployment的控制器资源清单，然后创建一个Service资源对象清单，通过Pod的Labels进行关联，然后通过Ingress或者Service的NodePort进行暴露服务，每次都需要这样操作，的确是略嫌麻烦，这个时候我们就可以自定义一个资源对象，通过CRD来描述需要部署的应用，信息，比如：镜像，服务端口，环境变量等等信息，然后创建自定义的资源的时候通过控制器去创建对应的Deploymnt，Service，Ingress等资源，这样貌似很方便，相当于我们将复杂化的资源清单进行了简单化，实现一个资源清单就可以做到很多个资源清单可以做的事情了。
```

```yaml
apiVersion: apps.kudevops.io/v1alpha1
kind: AppService
metadata:
  name: nginx
spec:
  replicas: 3
  image: nginx:alpine
  ports:
  - name: http
    port: 80
    targetPort: 80
    nodePort: 30080
```

```shell
如上所示就是我们的简单的自定义了一个资源对象，主要的参数就是定义了副本数量，Pod镜像，以及Service的信息，然后我们需要做的就是一步步的去实现这个Operator应用
```

```shell
# 开发环境
需要开发Operator自然Kubernets集群是少不了的，还需要有Golang环境，这里安装没什么可说的，然后我们需要安装operator-sdk，operator-sdk安装的方法非常多，可以直接在github上下载要使用的版本，然后放置到PATH环境下即可，当然也可以使用源码自己去编译安装也是完全OK的，我这里直接选择二进制了

# 因为我的集群是1.25.0的版本，所以我选择的operator-sdk的版本也不会太靠前
[root@localhost ~]# wget https://github.com/operator-framework/operator-sdk/releases/download/v1.28.0/operator-sdk_linux_amd64
[root@localhost ~]# mv operator-sdk_linux_amd64 /usr/local/bin/operator-sdk
[root@localhost ~]# chmod +x /usr/local/bin/operator-sdk 
[root@localhost ~]# operator-sdk version
[root@localhost app-operator]# operator-sdk version
operator-sdk version: "v1.28.0", commit: "484013d1865c35df2bc5dfea0ab6ea6b434adefa", kubernetes version: "1.26.0", go version: "go1.19.6", GOOS: "linux", GOARCH: "amd64"

# 创建项目
[root@localhost ~]# mkdir app-operator
[root@localhost ~]# cd app-operator/
[root@localhost app-operator]# operator-sdk init --domain kudevops.io --repo github.com/gitlayzer/app-operator
# 此处的--domain就是我们apiVersion中的apps后面的域名，可以理解为是一个group的名称，后面的repo就是package的名称，第一次如果没有下载包的话，需要等待一段时间，如果不是第一次的话会很快。
[root@localhost app-operator]# operator-sdk init --domain kudevops.io --repo github.com/gitlayzer/app-operator
Writing kustomize manifests for you to edit...
Writing scaffold for you to edit...
Get controller runtime:
$ go get sigs.k8s.io/controller-runtime@v0.14.1
Update dependencies:
$ go mod tidy
Next: define a resource with:
$ operator-sdk create api

# 创建完成之后，这里会让我们创建一个api，我们先看看当前的目录结构是什么样的
[root@localhost app-operator]# tree -L 2
.
├── config                            # 项目所需的所有文件
│   ├── default
│   ├── manager
│   ├── manifests
│   ├── prometheus
│   ├── rbac
│   └── scorecard
├── Dockerfile                        # 构建镜像
├── go.mod
├── go.sum
├── hack
│   └── boilerplate.go.txt
├── main.go                           # 整体项目入口文件
├── Makefile                          # 操作脚本
├── PROJECT
└── README.md

8 directories, 8 files

# 接下来就是按照上面所说，我们去创建一个API
[root@localhost app-operator]# operator-sdk create api --group apps --version v1alpha1 --kind AppService --resource --controller
Writing kustomize manifests for you to edit...
Writing scaffold for you to edit...
api/v1alpha1/appservice_types.go
controllers/appservice_controller.go
Update dependencies:
$ go mod tidy
Running make:
$ make generate
mkdir -p /root/app-operator/bin
test -s /root/app-operator/bin/controller-gen && /root/app-operator/bin/controller-gen --version | grep -q v0.11.1 || \
GOBIN=/root/app-operator/bin go install sigs.k8s.io/controller-tools/cmd/controller-gen@v0.11.1
go: downloading sigs.k8s.io/controller-tools v0.11.1
go: downloading github.com/spf13/cobra v1.6.1
go: downloading github.com/gobuffalo/flect v0.3.0
go: downloading golang.org/x/tools v0.4.0
go: downloading github.com/fatih/color v1.13.0
go: downloading k8s.io/utils v0.0.0-20221107191617-1a15be271d1d
go: downloading github.com/mattn/go-colorable v0.1.9
go: downloading github.com/mattn/go-isatty v0.0.14
go: downloading golang.org/x/net v0.4.0
go: downloading golang.org/x/mod v0.7.0
/root/app-operator/bin/controller-gen object:headerFile="hack/boilerplate.go.txt" paths="./..."
Next: implement your new API and generate the manifests (e.g. CRDs,CRs) with:
$ make manifests

# 从这里就可以看出来，它已经帮我们创建好了，我们再来看看新的目录结构
[root@localhost app-operator]# tree -L 2
.
├── api
│   └── v1alpha1
├── bin
│   └── controller-gen
├── config
│   ├── crd
│   ├── default
│   ├── manager
│   ├── manifests
│   ├── prometheus
│   ├── rbac
│   ├── samples
│   └── scorecard
├── controllers
│   ├── appservice_controller.go
│   └── suite_test.go
├── Dockerfile
├── go.mod
├── go.sum
├── hack
│   └── boilerplate.go.txt
├── main.go
├── Makefile
├── PROJECT
└── README.md

14 directories, 11 files

# 随后我们就需要去定义自己的结构体了，具体代码如下
```

```go
// api/v1alpha1/appservice_types.go
/*
Copyright 2023.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
*/

package v1alpha1

import (
	appsv1 "k8s.io/api/apps/v1"
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// EDIT THIS FILE!  THIS IS SCAFFOLDING FOR YOU TO OWN!
// NOTE: json tags are required.  Any new fields you add must have json tags for the fields to be serialized.

// AppServiceSpec defines the desired state of AppService
type AppServiceSpec struct {
  // INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
  // Important: Run "make" to regenerate code after modifying this file
  Replicas   *int32               `json:"replicas"`
  Image      string               `json:"image"`
  Ports      []corev1.ServicePort `json:"ports,omitempty"`
}

// AppServiceStatus defines the observed state of AppService
type AppServiceStatus struct {
  // INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
  // Important: Run "make" to regenerate code after modifying this file
  appsv1.DeploymentStatus `json:",inline"`
}

//+kubebuilder:object:root=true
//+kubebuilder:subresource:status

// AppService is the Schema for the appservices API
type AppService struct {
  metav1.TypeMeta   `json:",inline"`
  metav1.ObjectMeta `json:"metadata,omitempty"`

  Spec   AppServiceSpec   `json:"spec,omitempty"`
  Status AppServiceStatus `json:"status,omitempty"`
}

//+kubebuilder:object:root=true

// AppServiceList contains a list of AppService
type AppServiceList struct {
  metav1.TypeMeta `json:",inline"`
  metav1.ListMeta `json:"metadata,omitempty"`
  Items           []AppService `json:"items"`
}

func init() {
  SchemeBuilder.Register(&AppService{}, &AppServiceList{})
}
```

```shell
# 这些写完之后我们直接make以下命令，它会自动帮我们去go mod tidy 和生成相关的代码
[root@localhost app-operator]# make
test -s /root/app-operator/bin/controller-gen && /root/app-operator/bin/controller-gen --version | grep -q v0.11.1 || \
GOBIN=/root/app-operator/bin go install sigs.k8s.io/controller-tools/cmd/controller-gen@v0.11.1
/root/app-operator/bin/controller-gen rbac:roleName=manager-role crd webhook paths="./..." output:crd:artifacts:config=config/crd/bases
/root/app-operator/bin/controller-gen object:headerFile="hack/boilerplate.go.txt" paths="./..."
go fmt ./...
api/v1alpha1/appservice_types.go
go vet ./...
go build -o bin/manager main.go

# 当然这个时候如果我们还可以使用 make generate生成crd
[root@localhost app-operator]# make generate
test -s /root/app-operator/bin/controller-gen && /root/app-operator/bin/controller-gen --version | grep -q v0.11.1 || \
GOBIN=/root/app-operator/bin go install sigs.k8s.io/controller-tools/cmd/controller-gen@v0.11.1
/root/app-operator/bin/controller-gen object:headerFile="hack/boilerplate.go.txt" paths="./..."

# 我们来看看crd
[root@localhost app-operator]# ls config/crd/bases/apps.kudevops.io_appservices.yaml 
config/crd/bases/apps.kudevops.io_appservices.yaml

# 这个时候它就可以将CRD部署到K8S上去了
[root@localhost app-operator]# kubectl apply -k config/crd
customresourcedefinition.apiextensions.k8s.io/appservices.apps.kudevops.io created
# 查看部署的CRD
[root@localhost app-operator]# kubectl get crds | grep kudevops.io
appservices.apps.kudevops.io                          2023-06-14T13:54:17Z
# 查看支持的资源
[root@localhost app-operator]# kubectl api-resources | grep kudevops.io
appservices                 apps.kudevops.io/v1alpha1              true         AppService

# 下面我们就是需要去完善我们的自定义资源的清单了
```

```yaml
apiVersion: apps.kudevops.io/v1alpha1
kind: AppService
metadata:
  labels:
    app.kubernetes.io/name: appservice
    app.kubernetes.io/instance: appservice-sample
    app.kubernetes.io/part-of: app-operator
    app.kubernetes.io/managed-by: kustomize
    app.kubernetes.io/created-by: app-operator
  name: appservice-sample
spec:
  replicas: 3
  image: nginx:latest
  ports:
  - name: http
    port: 80
    targetPort: 80
    nodePort: 30080
```

```shell
# 我们部署一下资源
[root@localhost app-operator]# kubectl apply -f config/samples/apps_v1alpha1_appservice.yaml 
appservice.apps.kudevops.io/appservice-sample created、

# 查看资源
[root@localhost app-operator]# kubectl get appservices
NAME                AGE
appservice-sample   8s

# 但这个时候就像我们前面讲的CRD一样，没什么意义，因为我们还没有针对这个自定义资源去写控制器来对这个资源进行任何操作，那么接下来就是去写一个controller来针对这个自定义资源进行操作了。
```

```go
/*
Copyright 2023.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
*/

package controllers

import (
	"context"
	appsv1 "k8s.io/api/apps/v1"
	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/api/errors"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime/schema"

	"k8s.io/apimachinery/pkg/runtime"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/log"

	appsv1alpha1 "github.com/gitlayzer/app-operator/api/v1alpha1"
)

// AppServiceReconciler reconciles a AppService object
type AppServiceReconciler struct {
	client.Client
	Scheme *runtime.Scheme
}

//+kubebuilder:rbac:groups=apps.kudevops.io,resources=appservices,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=apps.kudevops.io,resources=appservices/status,verbs=get;update;patch
//+kubebuilder:rbac:groups=apps.kudevops.io,resources=appservices/finalizers,verbs=update

// Reconcile is part of the main kubernetes reconciliation loop which aims to
// move the current state of the cluster closer to the desired state.
// TODO(user): Modify the Reconcile function to compare the state specified by
// the AppService object against the actual cluster state, and then
// perform operations to make the cluster state reflect the state specified by
// the user.
//
// For more details, check Reconcile and its Result here:
// - https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.14.1/pkg/reconcile
func (r *AppServiceReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	reqLogger := log.Log.WithValues("Request.Namespace", req.Namespace, "Request.Name", req.Name)
	reqLogger.Info("Reconciling AppService......")

	instance := &appsv1alpha1.AppService{}
	if err := r.Client.Get(ctx, req.NamespacedName, instance); err != nil && errors.IsNotFound(err) {
		return ctrl.Result{}, nil
	}

	deploy := &appsv1.Deployment{}
	if err := r.Client.Get(ctx, req.NamespacedName, deploy); err != nil && errors.IsNotFound(err) {
		deployment := NewDeploy(instance)
		if err := r.Client.Create(ctx, deployment); err != nil {
			return ctrl.Result{}, err
		}

		service := NewService(instance)
		if err := r.Client.Create(ctx, service); err != nil {
			return ctrl.Result{}, err
		}
	}

	return ctrl.Result{}, nil
}

// SetupWithManager sets up the controller with the Manager.
func (r *AppServiceReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&appsv1alpha1.AppService{}).
		Complete(r)
}

func NewDeploy(app *appsv1alpha1.AppService) *appsv1.Deployment {
	return &appsv1.Deployment{
		TypeMeta: metav1.TypeMeta{
			Kind:       "Deployment",
			APIVersion: "apps/v1",
		},
		ObjectMeta: metav1.ObjectMeta{
			Name:      app.Name,
			Namespace: app.Namespace,
			OwnerReferences: []metav1.OwnerReference{
				*metav1.NewControllerRef(app, schema.GroupVersionKind{
					Group:   appsv1alpha1.GroupVersion.Group,
					Version: appsv1alpha1.GroupVersion.Version,
					Kind:    "AppService",
				}),
			},
		},
		Spec: appsv1.DeploymentSpec{
			Replicas: app.Spec.Replicas,
			Template: corev1.PodTemplateSpec{
				ObjectMeta: metav1.ObjectMeta{
					Labels: map[string]string{
						"app": app.Name,
					},
				},
				Spec: corev1.PodSpec{
					Containers: NewContainers(app),
				},
			},
			Selector: &metav1.LabelSelector{
				MatchLabels: map[string]string{
					"app": app.Name,
				},
			},
		},
	}
}

func NewContainers(app *appsv1alpha1.AppService) []corev1.Container {
	var containerPorts []corev1.ContainerPort
	for _, port := range app.Spec.Ports {
		containerPort := corev1.ContainerPort{}
		containerPort.ContainerPort = port.TargetPort.IntVal
		containerPorts = append(containerPorts, containerPort)
	}
	return []corev1.Container{{
		Name:            app.Name,
		Image:           app.Spec.Image,
		Ports:           containerPorts,
		ImagePullPolicy: corev1.PullIfNotPresent,
	}}
}

func NewService(app *appsv1alpha1.AppService) *corev1.Service {
	return &corev1.Service{
		TypeMeta: metav1.TypeMeta{
			Kind:       "Service",
			APIVersion: "v1",
		},
		ObjectMeta: metav1.ObjectMeta{
			Name:      app.Name,
			Namespace: app.Namespace,
			OwnerReferences: []metav1.OwnerReference{
				*metav1.NewControllerRef(app, schema.GroupVersionKind{
					Group:   appsv1alpha1.GroupVersion.Group,
					Version: appsv1alpha1.GroupVersion.Version,
					Kind:    "AppService",
				}),
			},
		},
		Spec: corev1.ServiceSpec{
			Type:  corev1.ServiceTypeNodePort,
			Ports: app.Spec.Ports,
			Selector: map[string]string{
				"app": app.Name,
			},
		},
	}
}
```

```shell
# 如上图所示，这就是一个最简单的控制器，我们只创建资源，不涉及其他的操作，那么下面就是针对这个控制器进行调试了，我们可以直接使用make脚本部署这个控制器
[root@localhost app-operator]# make
/root/app-operator/bin/controller-gen rbac:roleName=manager-role crd webhook paths="./..." output:crd:artifacts:config=config/crd/bases
/root/app-operator/bin/controller-gen object:headerFile="hack/boilerplate.go.txt" paths="./..."
go fmt ./...
go vet ./...
go build -o bin/manager main.go
[root@localhost app-operator]# make generate
test -s /root/app-operator/bin/controller-gen && /root/app-operator/bin/controller-gen --version | grep -q v0.11.1 || \
GOBIN=/root/app-operator/bin go install sigs.k8s.io/controller-tools/cmd/controller-gen@v0.11.1
/root/app-operator/bin/controller-gen object:headerFile="hack/boilerplate.go.txt" paths="./..."
[root@localhost app-operator]# make install run
test -s /root/app-operator/bin/controller-gen && /root/app-operator/bin/controller-gen --version | grep -q v0.11.1 || \
GOBIN=/root/app-operator/bin go install sigs.k8s.io/controller-tools/cmd/controller-gen@v0.11.1
/root/app-operator/bin/controller-gen rbac:roleName=manager-role crd webhook paths="./..." output:crd:artifacts:config=config/crd/bases
test -s /root/app-operator/bin/kustomize || { curl -Ss "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash -s -- 3.8.7 /root/app-operator/bin; }
/root/app-operator/bin/kustomize build config/crd | kubectl apply -f -
customresourcedefinition.apiextensions.k8s.io/appservices.apps.kudevops.io created
/root/app-operator/bin/controller-gen object:headerFile="hack/boilerplate.go.txt" paths="./..."
go fmt ./...
go vet ./...
go run ./main.go
2023-06-14T23:52:34+08:00	INFO	controller-runtime.metrics	Metrics server is starting to listen	{"addr": ":8080"}
2023-06-14T23:52:34+08:00	INFO	setup	starting manager
2023-06-14T23:52:34+08:00	INFO	Starting server	{"path": "/metrics", "kind": "metrics", "addr": "[::]:8080"}
2023-06-14T23:52:34+08:00	INFO	Starting server	{"kind": "health probe", "addr": "[::]:8081"}
2023-06-14T23:52:34+08:00	INFO	Starting EventSource	{"controller": "appservice", "controllerGroup": "apps.kudevops.io", "controllerKind": "AppService", "source": "kind source: *v1alpha1.AppService"}
2023-06-14T23:52:34+08:00	INFO	Starting Controller	{"controller": "appservice", "controllerGroup": "apps.kudevops.io", "controllerKind": "AppService"}
2023-06-14T23:52:34+08:00	INFO	Starting workers	{"controller": "appservice", "controllerGroup": "apps.kudevops.io", "controllerKind": "AppService", "worker count": 1}

# 这个过程中是很尿性的，它会去下载kustomize，但是仔细观察一下它的makefile脚本就可以看到它在判断./bin下面是否有这个程序，我们如果本地有直接链接过来或者改改makefile脚本就行了

# 这个时候可以看到控制器已经启动了，并且日志都给我们打印出来了，当我删除资源的时候就会触发调协
[root@localhost app-operator]# kubectl delete appservice appservice-sample
appservice.apps.kudevops.io "appservice-sample" deleted

# 这是调协打印的日志
2023-06-14T23:07:54+08:00	INFO	Reconciling AppService......	{"Request.Namespace": "default", "Request.Name": "appservice-sample"}
2023-06-14T23:09:35+08:00	INFO	Reconciling AppService......	{"Request.Namespace": "default", "Request.Name": "appservice-sample"}

# 然后我们去创建一个新的资源
[root@localhost app-operator]# kubectl apply -k config/samples/
appservice.apps.kudevops.io/appservice-sample created

# 可以看到调协的日志又来了
2023-06-14T23:11:21+08:00	INFO	Reconciling AppService......	{"Request.Namespace": "default", "Request.Name": "appservice-sample"}

# 然后跟着我们去看看根据我们的逻辑，是否创建了deployment和service
[root@localhost samples]# kubectl get deployment,svc
NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/appservice-sample   3/3     3            3           13s

NAME                        TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)        AGE
service/appservice-sample   NodePort    10.96.2.30   <none>        80:30080/TCP   13s
service/kubernetes          ClusterIP   10.96.0.1    <none>        443/TCP        44d

# 测试Pod是否可以访问
[root@localhost samples]# curl 10.0.0.11:30080 -I
HTTP/1.1 200 OK
Server: nginx/1.25.0
Date: Wed, 14 Jun 2023 15:54:05 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 23 May 2023 15:08:20 GMT
Connection: keep-alive
ETag: "646cd6e4-267"
Accept-Ranges: bytes

# 然后我们来测试一下关联删除，当我们删除我们自己的资源的时候，是否会跟着删除deployment和service
[root@k-m-1 ~]# kubectl get pod,svc
NAME                                     READY   STATUS    RESTARTS   AGE
pod/appservice-sample-5784bc58bf-5wb6k   1/1     Running   0          8s
pod/appservice-sample-5784bc58bf-62hh5   1/1     Running   0          8s
pod/appservice-sample-5784bc58bf-xdbwp   1/1     Running   0          8s

NAME                        TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
service/appservice-sample   NodePort    10.96.1.149   <none>        80:30080/TCP   8s
service/kubernetes          ClusterIP   10.96.0.1     <none>        443/TCP        45d

# 删除自定义资源
[root@localhost app-operator]# kubectl delete -k config/samples/
appservice.apps.kudevops.io "appservice-sample" deleted

# 再次查看资源
[root@k-m-1 ~]# kubectl get pod,svc
NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   45d


# 到这里我们的一个自定义的Operator就写好了，那么这个时候其实我们还要知道一些东西，就是operator-sdk还帮我们创建了一些资源比如监控接口，RBAC，ServiceMonitor，这个时候就是我们就可以去封装我们的控制器了，
```

## Getting Started
You’ll need a Kubernetes cluster to run against. You can use [KIND](https://sigs.k8s.io/kind) to get a local cluster for testing, or run against a remote cluster.
**Note:** Your controller will automatically use the current context in your kubeconfig file (i.e. whatever cluster `kubectl cluster-info` shows).

### Running on the cluster
1. Install Instances of Custom Resources:

```sh
kubectl apply -f config/samples/
```

2. Build and push your image to the location specified by `IMG`:

```sh
make docker-build docker-push IMG=<some-registry>/app-operator:tag
```

3. Deploy the controller to the cluster with the image specified by `IMG`:

```sh
make deploy IMG=<some-registry>/app-operator:tag
```

### Uninstall CRDs
To delete the CRDs from the cluster:

```sh
make uninstall
```

### Undeploy controller
UnDeploy the controller from the cluster:

```sh
make undeploy
```

## Contributing
// TODO(user): Add detailed information on how you would like others to contribute to this project

### How it works
This project aims to follow the Kubernetes [Operator pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/).

It uses [Controllers](https://kubernetes.io/docs/concepts/architecture/controller/),
which provide a reconcile function responsible for synchronizing resources until the desired state is reached on the cluster.

### Test It Out
1. Install the CRDs into the cluster:

```sh
make install
```

2. Run your controller (this will run in the foreground, so switch to a new terminal if you want to leave it running):

```sh
make run
```

**NOTE:** You can also run this in one step by running: `make install run`

### Modifying the API definitions
If you are editing the API definitions, generate the manifests such as CRs or CRDs using:

```sh
make manifests
```

**NOTE:** Run `make --help` for more information on all potential `make` targets

More information can be found via the [Kubebuilder Documentation](https://book.kubebuilder.io/introduction.html)

## License

Copyright 2023.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

