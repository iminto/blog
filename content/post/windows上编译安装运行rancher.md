---
title: "Windows上编译安装运行rancher"
date: 2020-02-28T10:07:59+08:00
archives: "2020"
tags: [k8s]
author: waitfox@qq.com
---
rancher 是一个为DevOps团队提供的完整的Kubernetes与容器管理解决方案。rancher最大的优点就是安装部署方便，极大地简化了K8S的安装配置。在官网上，推荐的是使用docker方式安装rancher，这种方式隐藏了大量的细节。在网上搜了下现有的资料，几乎都是照抄官方文档，更没有在windows上安装rancher的先例。

rancher是用golang写的，跨平台问题不大，但也需要一些修改。正好最近要对rancher做二次开发，于是记录下了在windows上编译安装rancher的步骤。

### 1.修改源码

rancher要在windows上编译通过并运行，需要修改以下源码

main.go

```go
func run(cfg app.Config) error {
   logrus.Infof("Rancher version %s is starting", VERSION)
   logrus.Infof("Rancher arguments %+v", cfg)
   dump.GoroutineDumpOn(syscall.SIGUSR1, syscall.SIGILL)
   ctx := signals.SetupSignalHandler(context.Background())
```


改为如下，此处修改基本不会有副作用

```go
func run(cfg app.Config) error {
   logrus.Infof("Rancher version %s is starting", VERSION)
   logrus.Infof("Rancher arguments %+v", cfg)
   dump.GoroutineDumpOn(syscall.SIGILL, syscall.SIGILL)
   ctx := signals.SetupSignalHandler(context.Background())
```

然后屏蔽以下几个文件中相关syscall的处理：（这几处修改可能会导致K8S相关的功能带来影响，但影响未知.建议使用条件编译方式）

pkg/controllers/user/helm/common/common.go

```go
func JailCommand(cmd *exec.Cmd, jailPath string) (*exec.Cmd, error) {
   if os.Getenv("CATTLE_DEV_MODE") != "" {
      return cmd, nil
   } else {
      //cred, err := jailer.GetUserCred()
      //if err != nil {
      // return nil, errors.WithMessage(err, "get user cred error")
      //}
      //
      //cmd.SysProcAttr = &syscall.SysProcAttr{}
      //cmd.SysProcAttr.Credential = cred
      //cmd.SysProcAttr.Chroot = jailPath
      //cmd.Env = jailer.WhitelistEnvvars(cmd.Env)
      return cmd, nil
   }
}
```

pkg/controllers/management/node/utils.go 修改同理

```go
func buildCommand(nodeDir string, node *v3.Node, cmdArgs []string) (*exec.Cmd, error) {
   // In dev_mode, don't need jail or reference to jail in command
   if os.Getenv("CATTLE_DEV_MODE") != "" {
      env := initEnviron(nodeDir)
      command := exec.Command(nodeCmd, cmdArgs...)
      command.Env = env
      return command, nil
   }

   //cred, err := jailer.GetUserCred()
   //if err != nil {
   // return nil, errors.WithMessage(err, "get user cred error")
   //}

   command := exec.Command(nodeCmd, cmdArgs...)
   //command.SysProcAttr = &syscall.SysProcAttr{}
   //command.SysProcAttr.Credential = cred
   //command.SysProcAttr.Chroot = path.Join(jailer.BaseJailPath, node.Namespace)
   envvars := []string{
      nodeDirEnvKey + nodeDir,
      "PATH=/usr/bin:/var/lib/rancher/management-state/bin",
   }
   command.Env = jailer.WhitelistEnvvars(envvars)
   return command, nil
}
```

屏蔽jailer的处理

pkg/jailer/jailer.go
注释掉这个方法
```go
// GetUserCred looks up the user and provides it in syscall.Credential
//func GetUserCred() (*syscall.Credential, error) {
//
//}
```
修改一个依赖库里的文件 （比较正规的方式是在go mod中使用replace语法，而不是直接修改第三方package）

vendor/github.com/rancher/kontainer-engine/service/service.go

```go
#446行开始注释这段代码
//if os.Getenv("CATTLE_DEV_MODE") == "" {
// cred, err := getUserCred()
// if err != nil {
//    return "", errors.WithMessage(err, "get user cred error")
// }
//
// cmd.SysProcAttr = &syscall.SysProcAttr{}
// cmd.SysProcAttr.Credential = cred
// cmd.SysProcAttr.Chroot = "/opt/jail/driver-jail"
// cmd.Env = whitelistEnvvars([]string{"PATH=/usr/bin"})
//}
```

628行注释掉这个方法
```
// getUserCred looks up the user and provides it in syscall.Credential
//func getUserCred() (*syscall.Credential, error) {
```
然后编译即可生成exe文件。

### 2.运行
如果笔记本内存在8G以上，可以安装windows版本k8s或者minikube

由于我本机配置太低，所以使用了linux服务器上的配置。
```
#每次运行一定要执行这个命令，或者在环境变量里配置加下也可一劳永逸
set CATTLE_DEV_MODE=true
go build -mod=vendor
#生成可执行文件 rancher.exe 运行文件需要k8s 在服务器上安装k8s 然后将配置文件k3s.yaml下载到本地windows上，修改配置文件的ip为k8s所在服务器上的ip 
#修改完成后执行下面指令 其中：g:\data\k3s.yaml为配置文件所在本地路径
rancher.exe  --k8s-mode=external  --kubeconfig g:\data\k3s.yaml --no-cacerts=true
```
k3s.yaml是Linux服务器上的K8S配置文件。不要使用服务器上已经被rancher使用过的k8s，而是用一个崭新的K8S环境。

服务器上安装k8s可以参考这里：https://k3s.io/
```
#这一步可能需要翻墙
curl -sfL https://get.k3s.io | sh -
#Check for Ready node, takes maybe 30 seconds
k3s kubectl get node
```
新增一个用户试一下，可以新增成功


### 3.条件编译
条件编译不需要在原有文件内容上做修改，编译出的文件可以保证在linux上是完整的。

以修改pkg/controllers/user/helm/common/common.go 为例，将原文件重命名未common_linux.go，然后新建一个common_windows_amd64.go，里面是windows版本的源码。

然后编译即可。

看起来条件编译对源文件做了重命名操作，但是保证了linux上代码的完整，不会让windows上的修改导致linux上产生隐患。

golang条件编译的源里可以参考此处：https://www.jianshu.com/p/4bb03e67e7ae

### 4.问题
1.未测试更多高级功能，还未知。

