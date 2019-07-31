---
title: kubelet服务启动流程中，默认参数的填充机制
date: 2019-07-31 10:28:15
tags:
	- kubernetes
---

本文跟踪一下kubelet启动过程中，参数的默认值是如何注入的。

我们知道，为了启动kubelet服务，首先要构造kubelet的配置对象，即kubeletconfig.KubeletConfiguration结构体，

```
// NewKubeletCommand creates a *cobra.Command object with default parameters
func NewKubeletCommand() *cobra.Command {
	cleanFlagSet := pflag.NewFlagSet(componentKubelet, pflag.ContinueOnError)
	cleanFlagSet.SetNormalizeFunc(cliflag.WordSepNormalizeFunc)
	kubeletFlags := options.NewKubeletFlags()
	#返回一个带有默认值的kubelet配置对象
	kubeletConfig, err := options.NewKubeletConfiguration()
	// programmer error
```

```
// NewKubeletConfiguration will create a new KubeletConfiguration with default values
func NewKubeletConfiguration() (*kubeletconfig.KubeletConfiguration, error) {
	//构造了一个针对kubelet的Schema，schema是为了解决数据结构 序列化、反序列化
	//以及多版本对象的兼容和转换问题引入的一个概念；Schema会登记资源对象到类型、类型到
	//资源对象的双向映射；不同版本数据对象的转换函数；
	//不同类型的默认初始化函数（设置default值）；
	scheme, _, err := kubeletscheme.NewSchemeAndCodecs()
	if err != nil {
		return nil, err
	}
	
	//v1beta1版本的kubelet配置
	versioned := &v1beta1.KubeletConfiguration{}
	//设置各字段的默认值，大致的原理是：先拿到versioned的类型，根据该类型找到对应的初始
	//化函数，然后调用初始化函数。
	scheme.Default(versioned)
	
	//无版本的kubelet配置
	config := &kubeletconfig.KubeletConfiguration{}
	//用v1beta1版本的配置去初始化无版本的kubelet配置，这里其实没有理解为什么要绕这么一下？？
	if err := scheme.Convert(versioned, config, nil); err != nil {
		return nil, err
	}
	
	//设置其他配置项的默认值
	applyLegacyDefaults(config)
	return config, nil
}
```

构造kubelet schmea的函数如下：

```
// NewSchemeAndCodecs is a utility function that returns a Scheme and CodecFactory
// that understand the types in the kubeletconfig API group.
func NewSchemeAndCodecs() (*runtime.Scheme, *serializer.CodecFactory, error) {
	//new了一个空的Schema
	scheme := runtime.NewScheme()
	
	//调用了register.go中的注册函数，针对资源：
	//GroupVersion{Group: GroupName, Version: runtime.APIVersionInternal} 
	//注册了两个类型：KubeletConfiguration 和 SerializedNodeConfigSource
	if err := kubeletconfig.AddToScheme(scheme); err != nil {
		return nil, nil, err
	}
	
	//调用v1beta1下的注册函数，为scheme设置了结构体默认值初始化函数，该初始化函数
	//符合结构体 KubeletConfiguration各字段默认值的设置，
	//具体的代码在v1beta1下的defaults.go文件中
	if err := kubeletconfigv1beta1.AddToScheme(scheme); err != nil {
		return nil, nil, err
	}
	
	codecs := serializer.NewCodecFactory(scheme)
	return scheme, &codecs, nil
}
```

最下层的初始化函数如下：

```
func SetDefaults_KubeletConfiguration(obj *kubeletconfigv1beta1.KubeletConfiguration) {
	if obj.SyncFrequency == zeroDuration {
		obj.SyncFrequency = metav1.Duration{Duration: 1 * time.Minute}
	}
	if obj.FileCheckFrequency == zeroDuration {
		obj.FileCheckFrequency = metav1.Duration{Duration: 20 * time.Second}
	}
	if obj.HTTPCheckFrequency == zeroDuration {
		obj.HTTPCheckFrequency = metav1.Duration{Duration: 20 * time.Second}
	}
	if obj.Address == "" {
		obj.Address = "0.0.0.0"
	}
	if obj.Port == 0 {
		obj.Port = ports.KubeletPort
	}
	if obj.Authentication.Anonymous.Enabled == nil {
		obj.Authentication.Anonymous.Enabled = utilpointer.BoolPtr(false)
	}
	if obj.Authentication.Webhook.Enabled == nil {
		obj.Authentication.Webhook.Enabled = utilpointer.BoolPtr(true)
	}
	if obj.Authentication.Webhook.CacheTTL == zeroDuration {
		obj.Authentication.Webhook.CacheTTL = metav1.Duration{Duration: 2 * time.Minute}
	}
	if obj.Authorization.Mode == "" {
		obj.Authorization.Mode = kubeletconfigv1beta1.KubeletAuthorizationModeWebhook
	}
	if obj.Authorization.Webhook.CacheAuthorizedTTL == zeroDuration {
		obj.Authorization.Webhook.CacheAuthorizedTTL = metav1.Duration{Duration: 5 * time.Minute}
	}
	if obj.Authorization.Webhook.CacheUnauthorizedTTL == zeroDuration {
		obj.Authorization.Webhook.CacheUnauthorizedTTL = metav1.Duration{Duration: 30 * time.Second}
	}
	if obj.RegistryPullQPS == nil {
		obj.RegistryPullQPS = utilpointer.Int32Ptr(5)
	}
	if obj.RegistryBurst == 0 {
		obj.RegistryBurst = 10
	}
```

最后看下，scheme的具体代码

```
// Scheme defines methods for serializing and deserializing API objects, a type
// registry for converting group, version, and kind information to and from Go
// schemas, and mappings between Go schemas of different versions. A scheme is the
// foundation for a versioned API and versioned configuration over time.
//
// In a Scheme, a Type is a particular Go struct, a Version is a point-in-time
// identifier for a particular representation of that Type (typically backwards
// compatible), a Kind is the unique name for that Type within the Version, and a
// Group identifies a set of Versions, Kinds, and Types that evolve over time. An
// Unversioned Type is one that is not yet formally bound to a type and is promised
// to be backwards compatible (effectively a "v1" of a Type that does not expect
// to break in the future).
//
// Schemes are not expected to change at runtime and are only threadsafe after
// registration is complete.
type Scheme struct {
	// versionMap allows one to figure out the go type of an object with
	// the given version and name.
	/*
	kubernetes中的类型对象都实现了Object接口，如下：
	// Object interface must be supported by all API types registered with Scheme. Since objects in a scheme are
// expected to be serialized to the wire, the interface an Object must provide to the Scheme allows
// serializers to set the kind, version, and group the object is represented as. An Object may choose
// to return a no-op ObjectKindAccessor in cases where it is not expected to be serialized.
type Object interface {
	GetObjectKind() schema.ObjectKind
	DeepCopyObject() Object
}

通过该接口可以获取到某个对象关联的资源类型GroupVersionKind：所属的Group、版本以及资源名；
有了gvkToType和typeToGVK所提供的信息，就可以结局数据对象的序列化和反序列化问题。
	*/
	gvkToType map[schema.GroupVersionKind]reflect.Type

	// typeToGroupVersion allows one to find metadata for a given go object.
	// The reflect.Type we index by should *not* be a pointer.
	typeToGVK map[reflect.Type][]schema.GroupVersionKind

	// unversionedTypes are transformed without conversion in ConvertToVersion.
	unversionedTypes map[reflect.Type]schema.GroupVersionKind

	// unversionedKinds are the names of kinds that can be created in the context of any group
	// or version
	// TODO: resolve the status of unversioned types.
	unversionedKinds map[string]reflect.Type

	// Map from version and resource to the corresponding func to convert
	// resource field labels in that version to internal version.
	fieldLabelConversionFuncs map[schema.GroupVersionKind]FieldLabelConversionFunc

	// defaulterFuncs is an array of interfaces to be called with an object to provide defaulting
	// the provided object must be a pointer.
	//初始化函数，每种类型都有自己的初始化函数
	defaulterFuncs map[reflect.Type]func(interface{})

	// converter stores all registered conversion functions. It also has
	// default converting behavior.
	converter *conversion.Converter

	// versionPriority is a map of groups to ordered lists of versions for those groups indicating the
	// default priorities of these versions as registered in the scheme
	versionPriority map[string][]string

	// observedVersions keeps track of the order we've seen versions during type registration
	observedVersions []schema.GroupVersion

	// schemeName is the name of this scheme.  If you don't specify a name, the stack of the NewScheme caller will be used.
	// This is useful for error reporting to indicate the origin of the scheme.
	schemeName string
}
```

总的调用链路大概如下：
options.NewKubeletConfiguration() -> kubeletscheme.NewSchemeAndCodecs()(完成kubeletSchema的构建，包括类型的注册，初始化函数的注册) -> scheme.Default() -> SetDefaults_KubeletConfiguration()