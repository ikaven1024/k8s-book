# API-客户端

客户端请求写法例如：

```go
pods := client.Core().Pods(namespace).Get(podname)
```

下面对上面逐层解析。

### client

client实现了`internalclientset.Interface`接口。在`NewForConfig`中实现。下面我们以Core()为例子继续深扒。

```go
// pkg/client/clientset_generated/internalclientset/
type Interface interface {
	Discovery() discovery.DiscoveryInterface
	Core() unversionedcore.CoreInterface
	Authentication() unversionedauthentication.AuthenticationInterface
	Authorization() unversionedauthorization.AuthorizationInterface
	Autoscaling() unversionedautoscaling.AutoscalingInterface
	Batch() unversionedbatch.BatchInterface
	Certificates() unversionedcertificates.CertificatesInterface
	Extensions() unversionedextensions.ExtensionsInterface
	Rbac() unversionedrbac.RbacInterface
	Storage() unversionedstorage.StorageInterface
}

// pkg/client/clientset_generated/internalclientset/
func NewForConfig(c *restclient.Config) (*Clientset, error) {
    clientset.CoreClient, err = unversionedcore.NewForConfig(&configShallowCopy)
    ...
}
```

### Core

CoreClient是对RESTFull的一层包装，实现了`unversioned.CoreInterface`接口，这里提供了对各种资源的访问接口。下面以Pod资源为例

```go
// pkg/client/clientset_generated/internalclientset/typed/core/unversioned/
type CoreClient struct {
	*restclient.RESTClient
}

type RESTClient struct {
	Client *http.Client
}
```

### Pods

Pods()返回对pod资源的访问接口，接口支持Create/Delete/Get/List/Patch/Update/Watch等方法。

```go
// pkg/client/clientset_generated/internalclientset/typed/core/unversioned/
func (c *CoreClient) Pods(namespace string) PodInterface {
	return newPods(c, namespace)
}

// pkg/client/clientset_generated/internalclientset/typed/core/unversioned/
func newPods(c *CoreClient, namespace string) *pods {
	return &pods{
		client: c,
		ns:     namespace,
	}
}
```

### 资源方法

#### Create

```go
func (c *pods) Get(name string) (result *api.Pod, err error) {
	result = &api.Pod{}
	err = c.client.Get().
		Namespace(c.ns).
		Resource("pods").
		Name(name).
		Do().
		Into(result)
	return
}
```
`client.Get()`构造了一个`restclient.Request`，并填充Verb字段。Verb包括`PUT/POST/GET/DELETE/PATCH`。后面的Namespace, Resource, Name都是在填充字段。

```go
type Request struct {
	// required
	client HTTPClient
	verb   string

	baseURL     *url.URL
	content     ContentConfig
	serializers Serializers

	// generic components accessible via method setters
	pathPrefix string
	subpath    string
	params     url.Values
	headers    http.Header

	// structural elements of the request that are part of the Kubernetes API conventions
	namespace    string
	namespaceSet bool
	resource     string
	resourceName string
	subresource  string
	selector     labels.Selector
	timeout      time.Duration

	// output
	err  error
	body io.Reader

	// The constructed request and the response
	req  *http.Request
	resp *http.Response

	backoffMgr BackoffManager
	throttle   flowcontrol.RateLimiter
}
```



```go
func (r *Request) Do() Result {
	r.tryThrottle()    // 流量控制

	err := r.request(func(req *http.Request, resp *http.Response) {
		result = r.transformResponse(resp, req)
	})
	return result
}

func (r *Request) request(fn func(*http.Request, *http.Response)) error {
	maxRetries := 10  // 最大尝试次数
  	for {
    	resp, err := client.Do(req)
      	checkWait(resp)
        ...
    }
}
```

Do()最后从Response中获取结构化的结构，最后调用Into()方法转化为api.Pod结构。

资源的`Update/UpdateStatus/Delete/DeleteCollection/Get/List`都差不多的过程。

#### Watch

Watch操作与上面区别于：用Watch方法代替了上面的Do和Into。

```go
func NewStreamWatcher(d Decoder) *StreamWatcher {
	sw := &StreamWatcher{...}
	go sw.receive()  // 开启接受任务，将数据推送给sw.result通道
	return sw        // 可以访问sw.ResultChan来获取result通道，从而进行消费。
}
```

### Reflector

