---
title: "K8s管理crd和K8SAPI编程"
date: 2020-02-11T14:38:59+08:00
archives: "2020"
tags: [k8s]
author: waitfox@qq.com
---
k8s每个版本看起来兼容性不是太好，很多网上的例子跑起来往往都有问题。

目前用的版本

```
root@de001:/develop# kubectl version
Client Version: version.Info{Major:"1", Minor:"17", GitVersion:"v1.17.2+k3s1", GitCommit:"cdab19b09a84389ffbf57bebd33871c60b1d6b28", GitTreeState:"clean", BuildDate:"2020-01-27T18:09:26Z", GoVersion:"go1.13.6", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"17", GitVersion:"v1.17.2+k3s1", GitCommit:"cdab19b09a84389ffbf57bebd33871c60b1d6b28", GitTreeState:"clean", BuildDate:"2020-01-27T18:09:26Z", GoVersion:"go1.13.6", Compiler:"gc", Platform:"linux/amd64"}
```

### 1.编写Spec文档

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  # name must match the spec fields below, and be in the form: <plural>.<group>
  name: crontabs.chenwen.com
spec:
  # group name to use for REST API: /apis/<group>/<version>
  group: chenwen.com
  # list of versions supported by this CustomResourceDefinition
  versions:
  - name: v2
    # Each version can be enabled/disabled by Served flag.
    served: true
    # One and only one version must be marked as the storage version.
    storage: true
    # A schema is required
  # The conversion section is introduced in Kubernetes 1.13+ with a default value of
  # None conversion (strategy sub-field set to None).
  conversion:
    # None conversion assumes the same schema for all versions and only sets the apiVersion
    # field of custom resources to the proper value
    strategy: None
  # either Namespaced or Cluster
  scope: Namespaced
  names:
    # plural name to be used in the URL: /apis/<group>/<version>/<plural>
    plural: crontabs
    # singular name to be used as an alias on the CLI and for display
    singular: crontab
    # kind is normally the CamelCased singular type. Your resource manifests use this.
    kind: Crontab
    listKind: CrontabList
    # shortNames allow shorter string to match your resource on the CLI
    shortNames:
    - ct 
```

### 2.导入K8S

```bash
ln -s /etc/rancher/k3s/k3s.yaml ~/.kube/config
kubectl apply -f crontab_crd.yml
kubectl get crd
```

可以看到自己创建的crd了。

查看这个CRD

```
root@de001:/develop# kubectl describe crontabs.chenwen.com
Name:         my-test-crontab
Namespace:    default
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"chenwen.com/v2","kind":"Crontab","metadata":{"annotations":{},"name":"my-test-crontab","namespace":"default"},"spec":{"cron...
API Version:  chenwen.com/v2
Kind:         Crontab
Metadata:
  Creation Timestamp:  2020-02-05T06:09:10Z
  Generation:          1
  Resource Version:    19965
  Self Link:           /apis/chenwen.com/v2/namespaces/default/crontabs/my-test-crontab
  UID:                 27beca8a-0ddb-4861-8643-90bb2f850b0d
Spec:
  Cron Spec:  * * * * */10
  Image:      my-test-image
  Replicas:   2
Events:       <none>
root@de001:/develop# 
```

rancher启动的时候也会给K8S注册一些CRD

不太清楚这样创建的CRD里用rancher的API是否能看到（rancher环境未搭建好测试 ）

添加一个自定义对象

```yaml
apiVersion: chenwen.com/v2
kind: Crontab
metadata:
  name: my-test-crontab
spec:
  cronSpec: "* * * * */10"
  image: my-test-image
  replicas: 2
```

导入

```bash
kubectl apply -f test_crd.yml
kubectl get ct
#删除自定义对象
kubectl delete  ct my-test-crontab
#删除CRD
kubectl delete crd crontabs.chenwen.com
```

运行结果：

```bash
root@de001:/develop# kubectl get crd|grep cron
crontabs.chenwen.com      2020-02-05T06:08:53Z
root@de001:/develop# kubectl get ct
NAME              AGE
my-test-crontab   75s
root@de001:/develop# 
```

### 3.代码生成

先在gopath下建立如下目录

```
go
 └── src
      └── github.com
           └── examplechen
           		└── go.mod
           		└── hack
           		└── pkg
           			└── apis
    					└── chenwen.com
        					└── v1
            					├── doc.go
            					└── types.go
    
 └── pkg
 └── bin
```

然后安装https://github.com/kubernetes/code-generator 项目的代码到gopath下。

 doc.go文件内容

```go
// FileName: doc.go
// Distributed under terms of the GPL license.

// +k8s:deepcopy-gen=package

// Package v1 is the v1 version of the API.
// +groupName=chenwen.com
package v1 
```

上述代码中的两行注释，都是代码生成工具会用到的，一个是声明为整个v1包下的类型定义生成DeepCopy方法，另一个声明了这个包对应的API的组名，和CRD中的组名一致

“// +k8s:deepcopy-gen=package”：为这个package中的所有type生成deepcopy代码。

“// +groupName=crd.lijiaocn.com”：设置这个package对应的api group。

types.go文件内容

```go
package v1

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// +genclient
// +genclient:noStatus
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// CronTab is a top-level type. A client is created for it.
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
type Crontab struct {
	metav1.TypeMeta `json:",inline"`
	// +optional
	metav1.ObjectMeta `json:"metadata,omitempty"`

	// Username unique username of the consumer.
	Username string `json:"username,omitempty"`

	// CustomID existing unique ID for the consumer - useful for mapping
	// Kong with users in your existing database
	CustomID string `json:"custom_id,omitempty"`
	  // Spec is the custom resource spec
    Spec CrontabSpec `json:"spec"`
}

// the spec for a MyResource resource
type CrontabSpec struct {
    Min int `json:"min"`
}

// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
type CrontabList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata"`

	Items []Crontab `json:"items"`
}

// Configuration contains a plugin configuration
// +k8s:deepcopy-gen=false
type Configuration map[string]interface{} 
```

“// +genclient”：为该type生成client代码。

“// +genclient:noStatus”：为该type生成的client代码，不包含`UpdateStatus`方法。

“// +genclient:nonNamespaced”：如果是集群资源，设置为不带namespace。

还支持在注释中使用以下tag：

```
// +genclient:noVerbs
// +genclient:onlyVerbs=create,delete
// +genclient:skipVerbs=get,list,create,update,patch,delete,deleteCollection,watch
// +genclient:method=Create,verb=create,result=k8s.io/apimachinery/pkg/apis/meta/v1.Status
```

==CrontabSpec结构体的字段不需要和yaml文件里的Spec 部分一一对应==。

同级目录编写register.go (这个文件非必须，此文件的作用是通过addKnownTypes方法使得client可以知道Crontab类型的API对象)

```go
package v1

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/runtime/schema"

	examplecom "github.com/examplechen/pkg/apis/chenwen.com"
)

// SchemeGroupVersion is group version used to register these objects
var SchemeGroupVersion = schema.GroupVersion{Group: examplecom.GroupName, Version: "v1"}

// Resource takes an unqualified resource and returns a Group qualified GroupResource
func Resource(resource string) schema.GroupResource {
	return SchemeGroupVersion.WithResource(resource).GroupResource()
}

var (
	// localSchemeBuilder and AddToScheme will stay in k8s.io/kubernetes.
	SchemeBuilder      runtime.SchemeBuilder
	localSchemeBuilder = &SchemeBuilder
	AddToScheme        = localSchemeBuilder.AddToScheme
)

func init() {
	// We only register manually written functions here. The registration of the
	// generated functions takes place in the generated files. The separation
	// makes the code compile even when the generated files are missing.
	localSchemeBuilder.Register(addKnownTypes)
}

// Adds the list of known types to api.Scheme.
func addKnownTypes(scheme *runtime.Scheme) error {
	scheme.AddKnownTypes(SchemeGroupVersion,
		&Crontab{},
		&CrontabList{},
	)
	metav1.AddToGroupVersion(scheme, SchemeGroupVersion)
	return nil
}
```

go.mod文件内容

```
module github.com/examplechen

go 1.13
```

生成代码只需要以上三个文件和其对应的目录结构

生成代码

```bash
[koudai@koudai-pc v1]$ /develop/go/src/k8s.io/code-generator/generate-groups.sh  all github.com/examplechen/pkg/client/crontab     github.com/examplechen/pkg/apis chenwen.com:v1
Generating deepcopy funcs
Generating clientset for chenwen.com:v1 at github.com/examplechen/pkg/client/crontab/clientset
Generating listers for chenwen.com:v1 at github.com/examplechen/pkg/client/crontab/listers
Generating informers for chenwen.com:v1 at github.com/examplechen/pkg/client/crontab/informers
```

尤其需要注意的是generate-groups.sh必须是==绝对路径==，不能是进入到code-generator目录下执行相对路径，不然会报找不到包的报错。

另外，需要进入到**examplechen**目录执行，不然会报莫名其妙如下的错误

```bash
Generating deepcopy funcs
F0210 17:31:46.418659   16605 deepcopy.go:885] Hit an unsupported type invalid type for invalid type, from github.com/examplechen/pkg/apis/chenwen.com/v1.Crontab
```

生成后的目录树如下：

```bash
[koudai@koudai-pc examplechen]$ tree
.
├── go.mod
├── go.sum
├── hack
│   └── boilerplate.go.txt
└── pkg
    ├── apis
    │   └── chenwen.com
    │       ├── register.go
    │       └── v1
    │           ├── doc.go
    │           ├── types.go
    │           └── zz_generated.deepcopy.go
    └── client
        └── crontab
            ├── clientset
            │   └── versioned
            │       ├── clientset.go
            │       ├── doc.go
            │       ├── fake
            │       │   ├── clientset_generated.go
            │       │   ├── doc.go
            │       │   └── register.go
            │       ├── scheme
            │       │   ├── doc.go
            │       │   └── register.go
            │       └── typed
            │           └── chenwen.com
            │               └── v1
            │                   ├── chenwen.com_client.go
            │                   ├── crontab.go
            │                   ├── doc.go
            │                   ├── fake
            │                   │   ├── doc.go
            │                   │   ├── fake_chenwen.com_client.go
            │                   │   └── fake_crontab.go
            │                   └── generated_expansion.go
            ├── informers
            │   └── externalversions
            │       ├── chenwen.com
            │       │   ├── interface.go
            │       │   └── v1
            │       │       ├── crontab.go
            │       │       └── interface.go
            │       ├── factory.go
            │       ├── generic.go
            │       └── internalinterfaces
            │           └── factory_interfaces.go
            └── listers
                └── chenwen.com
                    └── v1
                        ├── crontab.go
                        └── expansion_generated.go

23 directories, 29 files
```

### 4.使用自动生成的代码

examplechen 下新建main.go用来测试

```go
package main

import (
	"flag"
	"fmt"

	"github.com/golang/glog"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"

	"k8s.io/client-go/tools/clientcmd"
    "k8s.io/client-go/rest"

	examplecomclientset "github.com/examplechen/pkg/client/crontab/clientset/versioned"
)

var (
	kuberconfig = flag.String("kubeconfig", "", "Path to a kubeconfig. Only required if out-of-cluster.")
	master      = flag.String("master", "", "The address of the Kubernetes API server. Overrides any value in kubeconfig. Only required if out-of-cluster.")
)

func main() {
	flag.Parse()

   cfg, err := buildConfig("https://127.0.0.1:6443", "/root/.kube/config")
   if err != nil {
      fmt.Printf("%v\n", err)
      return
   }

	exampleClient, err := examplecomclientset.NewForConfig(cfg)
	if err != nil {
		glog.Fatalf("Error building example clientset: %v", err)
	}
	list, err := exampleClient.ChenwenV1().Crontabs("default").List(metav1.ListOptions{})
	if err != nil {
		glog.Fatalf("Error listing all databases: %v", err)
	}
     for _, db := range list.Items {
		fmt.Printf("database %s with user %q\n", db.Name, db.Spec.Min)
	}
}

func buildConfig(master, kubeconfig string) (*rest.Config, error) {
   if master != "" || kubeconfig != "" {
      return clientcmd.BuildConfigFromFlags(master, kubeconfig)
   }
   return rest.InClusterConfig()
}
```

编译通过，但运行报错。

```bash
F0210 14:19:32.477583    1845 main.go:39] Error listing all databases: the server could not find the requested resource (get crontabs.chenwen.com)
```

经检查，是版本号不一致造成，将yml文件里的版本号v2换成v1，另一个yml文件同理

```yaml
spec:
  # group name to use for REST API: /apis/<group>/<version>
  group: chenwen.com
  # list of versions supported by this CustomResourceDefinition
  versions:
  - name: v1 # 把v2换成v1，需要和API对应
```

重新编译，运行结果如下，符合预期

```
#go build
#./examplechenold -kubeconfig=$HOME/.kube/config 
database my-test-crontab with user '\x00'
database my-test-crontab2 with user '\x00'
```

### 5.进一步了解API

再写个稍微复杂点的例子

在 项目根目录下新建controller.go文件

```go
package main

import (
	"fmt"
	"time"

	"github.com/golang/glog"
	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/api/errors"
	"k8s.io/apimachinery/pkg/util/runtime"
	utilruntime "k8s.io/apimachinery/pkg/util/runtime"
	"k8s.io/apimachinery/pkg/util/wait"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/kubernetes/scheme"
	typedcorev1 "k8s.io/client-go/kubernetes/typed/core/v1"
	"k8s.io/client-go/tools/cache"
	"k8s.io/client-go/tools/record"
	"k8s.io/client-go/util/workqueue"

	bolingcavalryv1 "github.com/examplechen/pkg/apis/chenwen.com/v1"
	clientset "github.com/examplechen/pkg/client/crontab/clientset/versioned"
	cronscheme "github.com/examplechen/pkg/client/crontab/clientset/versioned/scheme"
	informers "github.com/examplechen/pkg/client/crontab/informers/externalversions/chenwen.com/v1"
	listers "github.com/examplechen/pkg/client/crontab/listers/chenwen.com/v1"
)

const controllerAgentName = "student-controller"

const (
	SuccessSynced = "Synced"

	MessageResourceSynced = "Student synced successfully"
)

// Controller is the controller implementation for Student resources
type Controller struct {
	// kubeclientset is a standard kubernetes clientset
	kubeclientset kubernetes.Interface
	// cronclientset is a clientset for our own API group
	cronclientset clientset.Interface

	cronsLister listers.CrontabLister
	cronsSynced cache.InformerSynced

	workqueue workqueue.RateLimitingInterface

	recorder record.EventRecorder
}

// NewController returns a new student controller
func NewController(
	kubeclientset kubernetes.Interface,
	cronclientset clientset.Interface,
	cronInformer informers.CrontabInformer) *Controller {

	utilruntime.Must(cronscheme.AddToScheme(scheme.Scheme))
	glog.V(4).Info("Creating event broadcaster")
	eventBroadcaster := record.NewBroadcaster()
	eventBroadcaster.StartLogging(glog.Infof)
	eventBroadcaster.StartRecordingToSink(&typedcorev1.EventSinkImpl{Interface: kubeclientset.CoreV1().Events("")})
	recorder := eventBroadcaster.NewRecorder(scheme.Scheme, corev1.EventSource{Component: controllerAgentName})

	controller := &Controller{
		kubeclientset: kubeclientset,
		cronclientset: cronclientset,
		cronsLister:   cronInformer.Lister(),
		cronsSynced:   cronInformer.Informer().HasSynced,
		workqueue:     workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), "Students"),
		recorder:      recorder,
	}

	glog.Info("Setting up event handlers")
	// Set up an event handler for when Student resources change
	cronInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: controller.enqueueStudent,
		UpdateFunc: func(old, new interface{}) {
			oldStudent := old.(*bolingcavalryv1.Crontab)
			newStudent := new.(*bolingcavalryv1.Crontab)
			if oldStudent.ResourceVersion == newStudent.ResourceVersion {
				//版本一致，就表示没有实际更新的操作，立即返回
				return
			}
			controller.enqueueStudent(new)
		},
		DeleteFunc: controller.enqueueStudentForDelete,
	})

	return controller
}

//在此处开始controller的业务
func (c *Controller) Run(threadiness int, stopCh <-chan struct{}) error {
	defer runtime.HandleCrash()
	defer c.workqueue.ShutDown()

	glog.Info("开始controller业务，开始一次缓存数据同步")
	if ok := cache.WaitForCacheSync(stopCh, c.cronsSynced); !ok {
		return fmt.Errorf("failed to wait for caches to sync")
	}

	glog.Info("worker启动")
	for i := 0; i < threadiness; i++ {
		go wait.Until(c.runWorker, time.Second, stopCh)
	}

	glog.Info("worker已经启动")
	<-stopCh
	glog.Info("worker已经结束")

	return nil
}

func (c *Controller) runWorker() {
	for c.processNextWorkItem() {
	}
}

// 取数据处理
func (c *Controller) processNextWorkItem() bool {

	obj, shutdown := c.workqueue.Get()

	if shutdown {
		return false
	}

	// We wrap this block in a func so we can defer c.workqueue.Done.
	err := func(obj interface{}) error {
		defer c.workqueue.Done(obj)
		var key string
		var ok bool

		if key, ok = obj.(string); !ok {

			c.workqueue.Forget(obj)
			runtime.HandleError(fmt.Errorf("expected string in workqueue but got %#v", obj))
			return nil
		}
		// 在syncHandler中处理业务
		if err := c.syncHandler(key); err != nil {
			return fmt.Errorf("error syncing '%s': %s", key, err.Error())
		}

		c.workqueue.Forget(obj)
		glog.Infof("Successfully synced '%s'", key)
		return nil
	}(obj)

	if err != nil {
		runtime.HandleError(err)
		return true
	}

	return true
}

// 处理
func (c *Controller) syncHandler(key string) error {
	// Convert the namespace/name string into a distinct namespace and name
	namespace, name, err := cache.SplitMetaNamespaceKey(key)
	if err != nil {
		runtime.HandleError(fmt.Errorf("invalid resource key: %s", key))
		return nil
	}

	// 从缓存中取对象
	student, err := c.cronsLister.Crontabs(namespace).Get(name)
	if err != nil {
		// 如果Cron对象被删除了，就会走到这里，所以应该在这里加入执行
		if errors.IsNotFound(err) {
			glog.Infof("Student对象被删除，请在这里执行实际的删除业务: %s/%s ...", namespace, name)

			return nil
		}

		runtime.HandleError(fmt.Errorf("failed to list student by: %s/%s", namespace, name))

		return err
	}

	glog.Infof("这里是cron对象的期望状态: %#v ...", student)
	glog.Infof("实际状态是从业务层面得到的，此处应该去的实际状态，与期望状态做对比，并根据差异做出响应(新增或者删除)")

	c.recorder.Event(student, corev1.EventTypeNormal, SuccessSynced, MessageResourceSynced)
	return nil
}

// 数据先放入缓存，再入队列
func (c *Controller) enqueueStudent(obj interface{}) {
	var key string
	var err error
	// 将对象放入缓存
	if key, err = cache.MetaNamespaceKeyFunc(obj); err != nil {
		runtime.HandleError(err)
		return
	}

	// 将key放入队列
	c.workqueue.AddRateLimited(key)
}

// 删除操作
func (c *Controller) enqueueStudentForDelete(obj interface{}) {
	var key string
	var err error
	// 从缓存中删除指定对象
	key, err = cache.DeletionHandlingMetaNamespaceKeyFunc(obj)
	if err != nil {
		runtime.HandleError(err)
		return
	}
	//再将key放入队列
	c.workqueue.AddRateLimited(key)
}

```

然后是 main.go

```go
package main

import (
	"flag"
	"fmt"
	"time"
	"github.com/golang/glog"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/kubernetes"
	informers "github.com/examplechen/pkg/client/crontab/informers/externalversions"
	"github.com/examplechen/pkg/signals"

	examplecomclientset "github.com/examplechen/pkg/client/crontab/clientset/versioned"
)

var (
	kubeconfig string
	master  string
)

func main() {
	flag.Parse()
	// 处理信号量
	stopCh := signals.SetupSignalHandler()

   cfg, err := clientcmd.BuildConfigFromFlags(master, kubeconfig)
   if err != nil {
      fmt.Printf("%v\n", err)
      return
   }

	kubeClient, err := kubernetes.NewForConfig(cfg)
	if err != nil {
		glog.Fatalf("Error building kubernetes clientset: %s", err.Error())
	}


	exampleClient, err := examplecomclientset.NewForConfig(cfg)
	if err != nil {
		glog.Fatalf("Error building example clientset: %v", err)
	}
	studentInformerFactory := informers.NewSharedInformerFactory(exampleClient, time.Second*30)

	//得到controller
	controller := NewController(kubeClient, exampleClient,
		studentInformerFactory.Chenwen().V1().Crontabs())

	//启动informer
	go studentInformerFactory.Start(stopCh)

	//controller开始处理消息
	if err = controller.Run(2, stopCh); err != nil {
		glog.Fatalf("Error running controller: %s", err.Error())
	}
}
func init() {
	flag.StringVar(&kubeconfig, "kubeconfig", "", "Path to a kubeconfig. Only required if out-of-cluster.")
	flag.StringVar(&master, "master", "", "The address of the Kubernetes API server. Overrides any value in kubeconfig. Only required if out-of-cluster.")
}
```

编译运行

```bash
root@de001:/develop# ./examplechen -kubeconfig=$HOME/.kube/config 
I0210 17:40:22.161175   18552 controller.go:72] Setting up event handlers
I0210 17:40:22.161769   18552 controller.go:96] 开始controller业务，开始一次缓存数据同步
I0210 17:40:22.262540   18552 controller.go:101] worker启动
I0210 17:40:22.262616   18552 controller.go:106] worker已经启动
I0210 17:40:22.262693   18552 controller.go:181] 这里是student对象的期望状态: &v1.Crontab{TypeMeta:v1.TypeMeta{Kind:"Crontab", APIVersion:"chenwen.com/v1"}, ObjectMeta:v1.ObjectMeta{Name:"my-test-crontab2", GenerateName:"", Namespace:"default", SelfLink:"/apis/chenwen.com/v1/namespaces/default/crontabs/my-test-crontab2", UID:"ab5e00e6-88e4-4b7e-b587-f5797baec3eb", ResourceVersion:"29576", Generation:1, CreationTimestamp:v1.Time{Time:time.Time{wall:0x0, ext:63716923534, loc:(*time.Location)(0x1e56c60)}}, DeletionTimestamp:(*v1.Time)(nil), DeletionGracePeriodSeconds:(*int64)(nil), Labels:map[string]string(nil), Annotations:map[string]string{"kubectl.kubernetes.io/last-applied-configuration":"{\"apiVersion\":\"chenwen.com/v1\",\"kind\":\"Crontab\",\"metadata\":{\"annotations\":{},\"name\":\"my-test-crontab2\",\"namespace\":\"default\"},\"spec\":{\"cronSpec\":\"* * * * */10\",\"image\":\"my-test-image\",\"replicas\":2}}\n"}, OwnerReferences:[]v1.OwnerReference(nil), Finalizers:[]string(nil), ClusterName:"", ManagedFields:[]v1.ManagedFieldsEntry(nil)}, Username:"", CustomID:"", Spec:v1.CrontabSpec{Min:0}} ...
I0210 17:40:22.262988   18552 controller.go:182] 实际状态是从业务层面得到的，此处应该去的实际状态，与期望状态做对比，并根据差异做出响应(新增或者删除)
I0210 17:40:22.263039   18552 controller.go:145] Successfully synced 'default/my-test-crontab2'
I0210 17:40:22.263063   18552 controller.go:181] 这里是student对象的期望状态: &v1.Crontab{TypeMeta:v1.TypeMeta{Kind:"Crontab", APIVersion:"chenwen.com/v1"}, ObjectMeta:v1.ObjectMeta{Name:"my-test-crontab", GenerateName:"", Namespace:"default", SelfLink:"/apis/chenwen.com/v1/namespaces/default/crontabs/my-test-crontab", UID:"72036436-451a-4ec5-9851-bc27342faa5f", ResourceVersion:"29532", Generation:1, CreationTimestamp:v1.Time{Time:time.Time{wall:0x0, ext:63716923360, loc:(*time.Location)(0x1e56c60)}}, DeletionTimestamp:(*v1.Time)(nil), DeletionGracePeriodSeconds:(*int64)(nil), Labels:map[string]string(nil), Annotations:map[string]string{"kubectl.kubernetes.io/last-applied-configuration":"{\"apiVersion\":\"chenwen.com/v1\",\"kind\":\"Crontab\",\"metadata\":{\"annotations\":{},\"name\":\"my-test-crontab\",\"namespace\":\"default\"},\"spec\":{\"cronSpec\":\"* * * * */10\",\"image\":\"my-test-image\",\"replicas\":2}}\n"}, OwnerReferences:[]v1.OwnerReference(nil), Finalizers:[]string(nil), ClusterName:"", ManagedFields:[]v1.ManagedFieldsEntry(nil)}, Username:"", CustomID:"", Spec:v1.CrontabSpec{Min:0}} ...
I0210 17:40:22.263157   18552 controller.go:182] 实际状态是从业务层面得到的，此处应该去的实际状态，与期望状态做对比，并根据差异做出响应(新增或者删除)
I0210 17:40:22.263185   18552 controller.go:145] Successfully synced 'default/my-test-crontab'
I0210 17:40:22.265730   18552 event.go:278] Event(v1.ObjectReference{Kind:"Crontab", Namespace:"default", Name:"my-test-crontab", UID:"72036436-451a-4ec5-9851-bc27342faa5f", APIVersion:"chenwen.com/v1", ResourceVersion:"29532", FieldPath:""}): type: 'Normal' reason: 'Synced' Student synced successfully
I0210 17:40:22.265885   18552 event.go:278] Event(v1.ObjectReference{Kind:"Crontab", Namespace:"default", Name:"my-test-crontab2", UID:"ab5e00e6-88e4-4b7e-b587-f5797baec3eb", APIVersion:"chenwen.com/v1", ResourceVersion:"29576", FieldPath:""}): type: 'Normal' reason: 'Synced' Student synced successfully
I0210 17:41:00.324824   18552 controller.go:181] 这里是student对象的期望状态: &v1.Crontab{TypeMeta:v1.TypeMeta{Kind:"", APIVersion:""}, ObjectMeta:v1.ObjectMeta{Name:"my-test-crontab2", GenerateName:"", Namespace:"default", SelfLink:"/apis/chenwen.com/v1/namespaces/default/crontabs/my-test-crontab2", UID:"ab5e00e6-88e4-4b7e-b587-f5797baec3eb", ResourceVersion:"29811", Generation:2, CreationTimestamp:v1.Time{Time:time.Time{wall:0x0, ext:63716923534, loc:(*time.Location)(0x1e56c60)}}, DeletionTimestamp:(*v1.Time)(nil), DeletionGracePeriodSeconds:(*int64)(nil), Labels:map[string]string(nil), Annotations:map[string]string{"kubectl.kubernetes.io/last-applied-configuration":"{\"apiVersion\":\"chenwen.com/v1\",\"kind\":\"Crontab\",\"metadata\":{\"annotations\":{},\"name\":\"my-test-crontab2\",\"namespace\":\"default\"},\"spec\":{\"cronSpec\":\"* 5 * * */10\",\"image\":\"my-test-image\",\"replicas\":2}}\n"}, OwnerReferences:[]v1.OwnerReference(nil), Finalizers:[]string(nil), ClusterName:"", ManagedFields:[]v1.ManagedFieldsEntry(nil)}, Username:"", CustomID:"", Spec:v1.CrontabSpec{Min:0}} ...
```

### 6.验证controller

新开一个窗口连接到k8s环境，新建一个名为test2_crd.yml的文件，内容如下

```
apiVersion: chenwen.com/v1
kind: Crontab
metadata:
  name: my-test-crontab2
spec:
  cronSpec: "* 5 * * */10"
  image: my-test-image
  replicas: 2
```

执行命令

```bash
kubectl apply -f test2_crd.yml 
```

返回controller所在的控制台窗口，发现新输出了如下内容，可见新增Crontab对象的事件已经被controller监听并处理：

```bash
I0210 17:41:00.324974   18552 controller.go:182] 实际状态是从业务层面得到的，此处应该去的实际状态，与期望状态做对比，并根据差异做出响应(新增或者删除)
I0210 17:41:00.325006   18552 controller.go:145] Successfully synced 'default/my-test-crontab2'
I0210 17:41:00.332401   18552 event.go:278] Event(v1.ObjectReference{Kind:"Crontab", Namespace:"default", Name:"my-test-crontab2", UID:"ab5e00e6-88e4-4b7e-b587-f5797baec3eb", APIVersion:"chenwen.com/v1", ResourceVersion:"29811", FieldPath:""}): type: 'Normal' reason: 'Synced' Student synced successfully
```

接下来您也可以尝试修改和删除已有的Crontab对象，观察controller控制台的输出，确定是否已经监听到所有Crontab变化的事件.

### 7.总结

三步走：

1. 创建CRD（Custom Resource Definition），令k8s明白我们自定义的API对象；
2. 编写代码，将CRD的情况写入对应的代码中，然后通过自动代码生成工具，将controller之外的informer，client等内容较为固定的代码通过工具生成；
3. 编写controller，在里面判断实际情况是否达到了API对象的声明情况，如果未达到，就要进行实际业务处理，而这也是controller的通用做法；

实际要自己动手写的文件不多，就3-4个，但是理解起来比较难。

### 8.refer

https://blog.csdn.net/weixin_41806245/article/details/94451734

https://blog.csdn.net/aixiaoyang168/article/details/81875907

https://github.com/kubernetes/sample-controller/blob/master/README.md

https://blog.openshift.com/kubernetes-deep-dive-code-generation-customresources/

https://www.jianshu.com/p/4910da4c8285

https://blog.csdn.net/boling_cavalry/article/details/88924194
