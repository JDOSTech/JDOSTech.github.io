---
layout:     post
title:      "Kubernetes API Server源码分析"
subtitle:   "apiserver源码分析"
date:       2016-09-22 00:00:00
author:     "lvjiangzhao"
header-img: "img/tag-bg.jpg"
catalog: true
tags:
    - Kubernetes
    - apiserver
---

# apiserver源码分析
Kubernetes Version: v1.3.0

## 1. apiserver流程梳理

### 1.1 GET /api/v1/namespaces/{namespace}/pods/{name}的处理过程
curl -i -v  http://10.8.65.156:8080/api/v1/namespaces/default/pods/pod-multi-containers
k8s存储在etcd内的对象是api.Pod对象（无版本），通过不同版本的api（如api/v1）访问Pod对象时，最终输出为v1.Pod的json形式，处理过程如下：

1. http client访问/api/v1/pod/xyz，想要获取这个Pod的数据
2. 从etcd获取到api.Pod对象
3. api.Pod对象转换为v1.Pod对象
4. v1.Pod对象序列化为json或yaml文本
5. 文本通过http的response，返回给http client

![](/img/2016-09-22-kube-apiserver-source/apigroupversion.png)

### 1.2 APIGroupVersion结构体

```go
// APIGroupVersion is a helper for exposing rest.Storage objects as http.Handlers via go-restful
// It handles URLs of the form:
// /${storage_key}[/${object_name}]
// Where 'storage_key' points to a rest.Storage object stored in storage.
// This object should contain all parameterization necessary for running a particular API version
type APIGroupVersion struct {
       Storage map[string]rest.Storage

       Root string

       // GroupVersion is the external group version
       GroupVersion unversioned.GroupVersion

       // RequestInfoResolver is used to parse URLs for the legacy proxy handler.  Don't use this for anything else
       // TODO: refactor proxy handler to use sub resources
       RequestInfoResolver *RequestInfoResolver

       // OptionsExternalVersion controls the Kubernetes APIVersion used for common objects in the apiserver
       // schema like api.Status, api.DeleteOptions, and api.ListOptions. Other implementors may
       // define a version "v1beta1" but want to use the Kubernetes "v1" internal objects. If
       // empty, defaults to GroupVersion.
       OptionsExternalVersion *unversioned.GroupVersion

       Mapper meta.RESTMapper

       // Serializer is used to determine how to convert responses from API methods into bytes to send over
       // the wire.
       Serializer     runtime.NegotiatedSerializer
       ParameterCodec runtime.ParameterCodec

       Typer     runtime.ObjectTyper
       Creater   runtime.ObjectCreater
       Convertor runtime.ObjectConvertor
       Copier    runtime.ObjectCopier
       Linker    runtime.SelfLinker

       Admit   admission.Interface
       Context api.RequestContextMapper

       MinRequestTimeout time.Duration

       // SubresourceGroupVersionKind contains the GroupVersionKind overrides for each subresource that is
       // accessible from this API group version. The GroupVersionKind is that of the external version of
       // the subresource. The key of this map should be the path of the subresource. The keys here should
       // match the keys in the Storage map above for subresources.
       SubresourceGroupVersionKind map[string]unversioned.GroupVersionKind
}
```

*重要成员：*

- GroupVersion：包含类似'api/v1'这样的string，用于标识这个实例
- Serializer：对象序列化和反序列化器
- Convertor：相互转换任意api版本的对象，需要事先注册转换函数
- Storage：key存放对象的url，value是一个rest.Storage，用于对接etcd存储

![](/img/2016-09-22-kube-apiserver-source/api.Pod.png)

*k8s目前提供的API分组有：*
1. 核心组：/api/v1
2. 扩展组：/apis/extensions/v1beta1
3. 其他：autoscaling、abac等

### 1.3 APIServer函数调用过程梳理（从storage到restful转换过程）

![](/img/2016-09-22-kube-apiserver-source/process.png)

![](/img/2016-09-22-kube-apiserver-source/overview.png)

## 2. apiserver与etcd存储对接
三层封装：pkg/registry、pkg/storage、etcd/clientv3

*pkg\registry\pod\etcd\etcd.go*

```go
func NewStorage
store := &registry.Store{
       NewFunc:     func() runtime.Object { return &api.Pod{} },
       NewListFunc: newListFunc,
       KeyRootFunc: func(ctx api.Context) string {
              return registry.NamespaceKeyRootFunc(ctx, prefix)
       },
       KeyFunc: func(ctx api.Context, name string) (string, error) {
              return registry.NamespaceKeyFunc(ctx, prefix, name)
       },
       ObjectNameFunc: func(obj runtime.Object) (string, error) {
              return obj.(*api.Pod).Name, nil
       },
       PredicateFunc: func(label labels.Selector, field fields.Selector) generic.Matcher {
              return pod.MatchPod(label, field)
       },
       QualifiedResource:       api.Resource("pods"),
       DeleteCollectionWorkers: opts.DeleteCollectionWorkers,

       CreateStrategy:      pod.Strategy,
       UpdateStrategy:      pod.Strategy,
       DeleteStrategy:      pod.Strategy,
       ReturnDeletedObject: true,

       Storage: storageInterface,
}
```

```go
// Get retrieves the object from the storage. It is required to support Patch.
func (r *StatusREST) Get(ctx api.Context, name string) (runtime.Object, error) {
       return r.store.Get(ctx, name)
}
```

*pkg\registry\generic\registry\store.go*

```go
type Store struct {
       // Called to make a new object, should return e.g., &api.Pod{}
       NewFunc func() runtime.Object

       // Called to make a new listing object, should return e.g., &api.PodList{}
       NewListFunc func() runtime.Object

       // Used for error reporting
       QualifiedResource unversioned.GroupResource

       // Used for listing/watching; should not include trailing "/"
       KeyRootFunc func(ctx api.Context) string

       // Called for Create/Update/Get/Delete. Note that 'namespace' can be
       // gotten from ctx.
       KeyFunc func(ctx api.Context, name string) (string, error)

       // Called to get the name of an object
       ObjectNameFunc func(obj runtime.Object) (string, error)

       // Return the TTL objects should be persisted with. Update is true if this
       // is an operation against an existing object. Existing is the current TTL
       // or the default for this operation.
       TTLFunc func(obj runtime.Object, existing uint64, update bool) (uint64, error)

       // Returns a matcher corresponding to the provided labels and fields.
       PredicateFunc func(label labels.Selector, field fields.Selector) generic.Matcher

       // DeleteCollectionWorkers is the maximum number of workers in a single
       // DeleteCollection call.
       DeleteCollectionWorkers int

       // Called on all objects returned from the underlying store, after
       // the exit hooks are invoked. Decorators are intended for integrations
       // that are above storage and should only be used for specific cases where
       // storage of the value is not appropriate, since they cannot
       // be watched.
       Decorator rest.ObjectFunc
       // Allows extended behavior during creation, required
       CreateStrategy rest.RESTCreateStrategy
       // On create of an object, attempt to run a further operation.
       AfterCreate rest.ObjectFunc
       // Allows extended behavior during updates, required
       UpdateStrategy rest.RESTUpdateStrategy
       // On update of an object, attempt to run a further operation.
       AfterUpdate rest.ObjectFunc
       // Allows extended behavior during updates, optional
       DeleteStrategy rest.RESTDeleteStrategy
       // On deletion of an object, attempt to run a further operation.
       AfterDelete rest.ObjectFunc
       // If true, return the object that was deleted. Otherwise, return a generic
       // success status response.
       ReturnDeletedObject bool
       // Allows extended behavior during export, optional
       ExportStrategy rest.RESTExportStrategy

       // Used for all storage access functions
       Storage storage.Interface
}
```

```go
// Get retrieves the item from storage.
func (e *Store) Get(ctx api.Context, name string) (runtime.Object, error) {
       obj := e.NewFunc()
       key, err := e.KeyFunc(ctx, name)
       if err != nil {
              return nil, err
       }
       if err := e.Storage.Get(ctx, key, obj, false); err != nil {
              return nil, storeerr.InterpretGetError(err, e.QualifiedResource, name)
       }
       if e.Decorator != nil {
              if err := e.Decorator(obj); err != nil {
                     return nil, err
              }
       }
       return obj, nil
}
```

*pkg\storage\interfaces.go*

```go
// Interface offers a common interface for object marshaling/unmarshling operations and
// hides all the storage-related operations behind it.
type Interface interface {
       // Returns list of servers addresses of the underyling database.
       // TODO: This method is used only in a single place. Consider refactoring and getting rid
       // of this method from the interface.
       Backends(ctx context.Context) []string

       // Returns Versioner associated with this interface.
       Versioner() Versioner

       // Create adds a new object at a key unless it already exists. 'ttl' is time-to-live
       // in seconds (0 means forever). If no error is returned and out is not nil, out will be
       // set to the read value from database.
       Create(ctx context.Context, key string, obj, out runtime.Object, ttl uint64) error

       // Delete removes the specified key and returns the value that existed at that spot.
       // If key didn't exist, it will return NotFound storage error.
       Delete(ctx context.Context, key string, out runtime.Object, preconditions *Preconditions) error

       // Watch begins watching the specified key. Events are decoded into API objects,
       // and any items passing 'filter' are sent down to returned watch.Interface.
       // resourceVersion may be used to specify what version to begin watching,
       // which should be the current resourceVersion, and no longer rv+1
       // (e.g. reconnecting without missing any updates).
       Watch(ctx context.Context, key string, resourceVersion string, filter FilterFunc) (watch.Interface, error)

       // WatchList begins watching the specified key's items. Items are decoded into API
       // objects and any item passing 'filter' are sent down to returned watch.Interface.
       // resourceVersion may be used to specify what version to begin watching,
       // which should be the current resourceVersion, and no longer rv+1
       // (e.g. reconnecting without missing any updates).
       WatchList(ctx context.Context, key string, resourceVersion string, filter FilterFunc) (watch.Interface, error)

       // Get unmarshals json found at key into objPtr. On a not found error, will either
       // return a zero object of the requested type, or an error, depending on ignoreNotFound.
       // Treats empty responses and nil response nodes exactly like a not found error.
       Get(ctx context.Context, key string, objPtr runtime.Object, ignoreNotFound bool) error

       // GetToList unmarshals json found at key and opaque it into *List api object
       // (an object that satisfies the runtime.IsList definition).
       GetToList(ctx context.Context, key string, filter FilterFunc, listObj runtime.Object) error

       // List unmarshalls jsons found at directory defined by key and opaque them
       // into *List api object (an object that satisfies runtime.IsList definition).
       // The returned contents may be delayed, but it is guaranteed that they will
       // be have at least 'resourceVersion'.
       List(ctx context.Context, key string, resourceVersion string, filter FilterFunc, listObj runtime.Object) error

       GuaranteedUpdate(ctx context.Context, key string, ptrToType runtime.Object, ignoreNotFound bool, precondtions *Preconditions, tryUpdate UpdateFunc) error

       // Codec provides access to the underlying codec being used by the implementation.
       Codec() runtime.Codec
}
```

*pkg\storage\etcd3\store.go*

```go
// Get implements storage.Interface.Get.
func (s *store) Get(ctx context.Context, key string, out runtime.Object, ignoreNotFound bool) error {
       key = keyWithPrefix(s.pathPrefix, key)
       getResp, err := s.client.KV.Get(ctx, key)
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
       return decode(s.codec, s.versioner, kv.Value, out, kv.ModRevision)
}
```

*vendor\github.com\coreos\etcd\clientv3\kv.go*

```go
func (kv *kv) Get(ctx context.Context, key string, opts ...OpOption) (*GetResponse, error) {
       r, err := kv.Do(ctx, OpGet(key, opts...))
       return r.get, err
}
```

```go
func (kv *kv) Do(ctx context.Context, op Op) (OpResponse, error) {
       for {
              var err error
              switch op.t {
              // TODO: handle other ops
              case tRange:
                     var resp *pb.RangeResponse
                     r := &pb.RangeRequest{Key: op.key, RangeEnd: op.end, Limit: op.limit, Revision: op.rev, Serializable: op.serializable}
                     if op.sort != nil {
                            r.SortOrder = pb.RangeRequest_SortOrder(op.sort.Order)
                            r.SortTarget = pb.RangeRequest_SortTarget(op.sort.Target)
                     }

                     resp, err = kv.getRemote().Range(ctx, r)
                     if err == nil {
                            return OpResponse{get: (*GetResponse)(resp)}, nil
                     }
              case tPut:
                     var resp *pb.PutResponse
                     r := &pb.PutRequest{Key: op.key, Value: op.val, Lease: int64(op.leaseID)}
                     resp, err = kv.getRemote().Put(ctx, r)
                     if err == nil {
                            return OpResponse{put: (*PutResponse)(resp)}, nil
                     }
              case tDeleteRange:
                     var resp *pb.DeleteRangeResponse
                     r := &pb.DeleteRangeRequest{Key: op.key, RangeEnd: op.end}
                     resp, err = kv.getRemote().DeleteRange(ctx, r)
                     if err == nil {
                            return OpResponse{del: (*DeleteResponse)(resp)}, nil
                     }
              default:
                     panic("Unknown op")
              }

              if isHalted(ctx, err) {
                     return OpResponse{}, err
              }

              // do not retry on modifications
              if op.isWrite() {
                     go kv.switchRemote(err)
                     return OpResponse{}, err
              }

              if nerr := kv.switchRemote(err); nerr != nil {
                     return OpResponse{}, nerr
              }
       }
}
```

*vendor\github.com\coreos\etcd\etcdserver\etcdserverpb\rpc.pb.go*

```go
type KVClient interface {
       // Range gets the keys in the range from the store.
       Range(ctx context.Context, in *RangeRequest, opts ...grpc.CallOption) (*RangeResponse, error)
       // Put puts the given key into the store.
       // A put request increases the revision of the store,
       // and generates one event in the event history.
       Put(ctx context.Context, in *PutRequest, opts ...grpc.CallOption) (*PutResponse, error)
       // Delete deletes the given range from the store.
       // A delete request increase the revision of the store,
       // and generates one event in the event history.
       DeleteRange(ctx context.Context, in *DeleteRangeRequest, opts ...grpc.CallOption) (*DeleteRangeResponse, error)
       // Txn processes all the requests in one transaction.
       // A txn request increases the revision of the store,
       // and generates events with the same revision in the event history.
       // It is not allowed to modify the same key several times within one txn.
       Txn(ctx context.Context, in *TxnRequest, opts ...grpc.CallOption) (*TxnResponse, error)
       // Compact compacts the event history in etcd. User should compact the
       // event history periodically, or it will grow infinitely.
       Compact(ctx context.Context, in *CompactionRequest, opts ...grpc.CallOption) (*CompactionResponse, error)
}
```

```go
func (c *kVClient) Range(ctx context.Context, in *RangeRequest, opts ...grpc.CallOption) (*RangeResponse, error) {
       out := new(RangeResponse)
       err := grpc.Invoke(ctx, "/etcdserverpb.KV/Range", in, out, c.cc, opts...)
       if err != nil {
              return nil, err
       }
       return out, nil
}
```

![](/img/2016-09-22-kube-apiserver-source/storage.png)


## 3.如何新增一个API
go-restful：

- restful.Container，表示一个http rest服务对象，包括一组restful.WebService
- restful.WebService，由多个restful.Route组成，处理这些路径下所有支持的媒体类型（json，yaml等）
- restful.Route，实现路由绑定（http method，url和处理函数）


```go
// GenericAPIServer contains state for a Kubernetes cluster api server.
type GenericAPIServer struct {
       // "Inputs", Copied from Config
       ServiceClusterIPRange *net.IPNet
       ServiceNodePortRange  utilnet.PortRange
       cacheTimeout          time.Duration
       MinRequestTimeout     time.Duration

       mux                   apiserver.Mux
       MuxHelper             *apiserver.MuxHelper
       HandlerContainer      *restful.Container
       RootWebService        *restful.WebService
       ......
}
```

```go
// Container holds a collection of WebServices and a http.ServeMux to dispatch http requests.
// The requests are further dispatched to routes of WebServices using a RouteSelector
type Container struct {
       webServicesLock        sync.RWMutex
       webServices            []*WebService
       ServeMux               *http.ServeMux
       isRegisteredOnRoot     bool
       containerFilters       []FilterFunction
       doNotRecover           bool // default is false
       recoverHandleFunc      RecoverHandleFunction
       serviceErrorHandleFunc ServiceErrorHandleFunction
       router                 RouteSelector // default is a RouterJSR311, CurlyRouter is the faster alternative
       contentEncodingEnabled bool          // default is false
}
```

*pkg\apiserver\apiserver.go*

```go
// Adds a service to return the supported api versions at the legacy /api.
func AddApiWebService(s runtime.NegotiatedSerializer, container *restful.Container, apiPrefix string, getAPIVersionsFunc func(req *restful.Request) *unversioned.APIVersions) {
       // TODO: InstallREST should register each version automatically

       // Because in release 1.1, /api returns response with empty APIVersion, we
       // use StripVersionNegotiatedSerializer to keep the response backwards
       // compatible.
       ss := StripVersionNegotiatedSerializer{s}
       versionHandler := APIVersionHandler(ss, getAPIVersionsFunc)
       ws := new(restful.WebService)
       ws.Path(apiPrefix)
       ws.Doc("get available API versions")
       ws.Route(ws.GET("/").To(versionHandler).
              Doc("get available API versions").
              Operation("getAPIVersions").
              Produces(s.SupportedMediaTypes()...).
              Consumes(s.SupportedMediaTypes()...).
              Writes(unversioned.APIVersions{}))
       container.Add(ws)
```

### 4. APIServer处理binding
*pkg\registry\pod\etcd\etcd.go*

```go
// PodStorage includes storage for pods and all sub resources
type PodStorage struct {
       Pod         *REST
       Binding     *BindingREST
       Status      *StatusREST
       Log         *podrest.LogREST
       Proxy       *podrest.ProxyREST
       Exec        *podrest.ExecREST
       Attach      *podrest.AttachREST
       PortForward *podrest.PortForwardREST
}
```

```go
type BindingREST struct {
       store *registry.Store
}
```

```go
// Create ensures a pod is bound to a specific host.
func (r *BindingREST) Create(ctx api.Context, obj runtime.Object) (out runtime.Object, err error) {
       binding := obj.(*api.Binding)

       // TODO: move me to a binding strategy
       if errs := validation.ValidatePodBinding(binding); len(errs) != 0 {
              return nil, errs.ToAggregate()
       }

       err = r.assignPod(ctx, binding.Name, binding.Target.Name, binding.Annotations)
       out = &unversioned.Status{Status: unversioned.StatusSuccess}
       return
}
```

