---
title: k8s work with etcd
description: 
categories:
- k8s
tags:
- storage
---
*本文讨论以下问题：*
> - *k8s的REST API如何注册？*
> - *REST API的操作如何体现到Etcd？*

### api install
当APIServer开始创建时，CreateKubeAPIServer函数用于创建GenericAPIServer对象，并由这个对象负责APIServer的所有服务。CreateKubeAPIServer首先创建各种APIServer使用的Config对象，其中的一个是completedConfig。在completedConfig对象的创建方法中，APIServer完成了服务API的Path到Storage的映射操作。

在APIServer的代码中，API分为多类注册。这里我们以LegacyAPI为例。InstallLegacyAPI函数负责LegacyAPI的注册，其调用的NewLegacyRestStorage方法中我们可以看到APIServer设置的Path到Storage的映射：

*k8s.io/kubernetes/pkg/registry/core/rest/storage_core.go*
```
    persistentVolumeStorage, persistentVolumeStatusStorage := pvstore.NewREST(restOptionsGetter)

    restStorageMap := map[string]rest.Storage{
        "pods":             podStorage.Pod,
        "pods/attach":      podStorage.Attach,
        ......
        "persistentVolumes":             persistentVolumeStorage,
        ......
    }
```
我们可以看到，pod、persistentVolumes等这些k8s资源在这里注册了对应的REST接口。
### Store以及Storage接口
以persistentVolumeStorage为例，我们看NewRest的实现：
```
    store := &genericregistry.Store{
        Copier:            api.Scheme,
        NewFunc:           func() runtime.Object { return &api.PersistentVolume{} },
    
    ......
    store.CompleteWithOptions(options)
```
PersistentVolume storage的NewREST方法为PersistentVolume创建了Store对象。Store对象的结构如下：
*pkg/registry/generic/registry/store.go*
```
type Store struct {
    NewFunc func() runtime.Object
    Storage storage.Interface

```
其中，NewFunc实现了对象创建，Storage实现对象状态的底层存储。在PersistentVolume的Store对象中我们可以看到，NewFunc实现了简单的新对象创建，CompleteWithOptions补充了一些Store中的属性，其中重要的就是Storage接口。如下所示：
*k8s.io/apiserver/pkg/registry/generic/registry/store.go*
```
func (e *Store) CompleteWithOptions(options *generic.StoreOptions) error {

    if e.Storage == nil {
        e.Storage, e.DestroyFunc = opts.Decorator(
            e.Copier,
            opts.StorageConfig,
            e.WatchCacheSize,
            e.NewFunc(),
            prefix,
            keyFunc,
            e.NewListFunc,
            options.AttrFunc,
            triggerFunc,
        )
```
Store对象中实现了资源对象的Create、List、Get、Update、Watch等存取操作。以Get为例，我们可以看出Store对象对Storage属性Get接口的封装：
```
// Get retrieves the item from storage.
func (e *Store) Get(ctx genericapirequest.Context, name string, options *metav1.GetOptions) (runtime.Object, error) {
    obj := e.NewFunc()
    err := e.Storage.Get(ctx, key, options.ResourceVersion, obj, false);
```
在上述代码中，当Store对象的Get接口被调用时，调用链还将继续传递到Storage接口的Get方法。因而，当系统使用Store的各个接口操作资源对象时，对象的变化最终会通过Storage接口反应到底层存储中。

### Etcd接口
接下来，我们跟踪opts.StorageConfig，这个属性最终决定了Storage接口。这个属性最终需要追溯到APIServer创建之处的执行选项。

在创建时，APIServer需要首先准备好RunOptions，我们从中可以看到APIServer有一个Etcd属性，保存了后续Etcd操作的一些配置信息，如下所示：
*k8s.io/kubernetes/cmd/kube-apiserver/app/options/options.go*
```
func NewServerRunOptions() *ServerRunOptions {
    s := ServerRunOptions{
        GenericServerRunOptions: genericoptions.NewServerRunOptions(),
        Etcd:                 genericoptions.NewEtcdOptions(storagebackend.NewDefaultConfig(kubeoptions.DefaultEtcdPathPrefix, api.Scheme, nil))
```
而在createAPIExtensionsConfig函数中，我们可以看到genericConfig.RESTOptionsGetter使用了对应的etcdOptions。opts.StorageConfig的opts对象最终也来自于RESTOptionsGetter。因而，我们可以确定上述资源的Store使用的Storage接口最终来自于Etcd。

*/Users/LarryCK/gCode/kubernetes-master/cmd/kube-apiserver/app/apiextensions.go*

```
func createAPIExtensionsConfig(kubeAPIServerConfig genericapiserver.Config, externalInformers kubeexternalinformers.SharedInformerFactory, commandOptions *options.ServerRunOptions) (*apiextensionsapiserver.Config, error) {

    genericConfig.RESTOptionsGetter = &genericoptions.SimpleRestOptionsFactory{Options: etcdOptions
```
###Etcd3
我们可以进一步分析Etcd提供的Storage接口。
*k8s.io/apiserver/pkg/storage/storagebackend/factory/factory.go*
```
func Create(c storagebackend.Config) (storage.Interface, DestroyFunc, error) {
    switch c.Type {
    case storagebackend.StorageTypeETCD2:
        return newETCD2Storage(c)
    case storagebackend.StorageTypeUnset, storagebackend.StorageTypeETCD3:
        return newETCD3Storage(c)
    default:
        return nil, nil, fmt.Errorf("unknown storage type: %s", c.Type)
    }
}
```
可见，如果StorageType如果没有指定，默认使用ETCD3Storage。
*k8s.io/apiserver/pkg/storage/etcd3/store.go*文件 实现了ETCD3版本的Store对象，并提供了Storage接口。以Get函数为例：

```
// Get implements storage.Interface.Get.
func (s *store) Get(ctx context.Context, key string, resourceVersion string, out runtime.Object, ignoreNotFound bool) error {
    key = path.Join(s.pathPrefix, key)
    getResp, err := s.client.KV.Get(ctx, key, s.getOps...)
    if err != nil {
        return err
    }

    if len(getResp.Kvs) == 0 {
        if ignoreNotFound {
            return runtime.SetZeroValue(out)
        }
        return storage.NewKeyNotFoundError(key, 0)
    }
    kv := getResp.Kvs[0]

    data, _, err := s.transformer.TransformFromStorage(kv.Value, authenticatedDataString(key))
    if err != nil {
        return storage.NewInternalError(err.Error())
    }

    return decode(s.codec, s.versioner, data, out, kv.ModRevision)
}
```
该函数从ETCD获得对应的数据并且解压得到最终的对象内容。

至此，整个APIServer从api注册到Etcd后端数据存取的流程已经解释清楚了。后续还有一些问题可以进一步探讨，例如：Etcd3使用codec是哪种？现在系统中提供的Serializer包括Json、Yaml、Protobuf等多种选择。


