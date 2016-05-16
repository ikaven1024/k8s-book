# APIServer



关键调用堆栈：

```go
func New(c *Config) (*Master, error)
..func (m *Master) InstallAPIs(c *Config)
....func (s *GenericAPIServer) InstallAPIGroups(groupsInfo []APIGroupInfo) error
......func (s *GenericAPIServer) installAPIGroup(apiGroupInfo *APIGroupInfo) error
func (s *GenericAPIServer) Run(options *ServerRunOptions)
```





```go
type Etcd struct {
	NewFunc func() runtime.Object		// 产生新的对象
	NewListFunc func() runtime.Object	// 产生新的对象列表

	// Used for error reporting
	QualifiedResource unversioned.GroupResource		// 

	// Used for listing/watching; should not include trailing "/"
	KeyRootFunc func(ctx api.Context) string

	// Called for Create/Update/Get/Delete. Note that 'namespace' can be
	// gotten from ctx.
	KeyFunc func(ctx api.Context, name string) (string, error)

	ObjectNameFunc func(obj runtime.Object) (string, error)		// 由object产生名字

	// Return the TTL objects should be persisted with. Update is true if this
	// is an operation against an existing object. Existing is the current TTL
	// or the default for this operation.
	TTLFunc func(obj runtime.Object, existing uint64, update bool) (uint64, error)

	PredicateFunc func(label labels.Selector, field fields.Selector) generic.Matcher

	// DeleteCollectionWorkers is the maximum number of workers in a single
	// DeleteCollection call.
	DeleteCollectionWorkers int

	// Called on all objects returned from the underlying store, after
	// the exit hooks are invoked. Decorators are intended for integrations
	// that are above etcd and should only be used for specific cases where
	// storage of the value in etcd is not appropriate, since they cannot
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

	// Used for all etcd access functions
	Storage storage.Interface
}
```



