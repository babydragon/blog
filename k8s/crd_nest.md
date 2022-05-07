# 从0开始学kubernetes——CRD套娃

前面尝试了写一个非常简单的CRD，有的时候需要很多controller之间配合拆解一些任务，因此尝试做一个CRD，实现通过其yaml中声明的类型来提交一个新的CR，让其他controller来接力执行。

还是基于之前的demo，再用`kubebuilder`来创建一个api：

```
kubebuilder create api --group web --version v1 --kind Web
```

这里创建一个Kind为`Web`的CRD，我们需要修改下自动生成的Spec结构体，以适配一些参数：

```go
// WebSpec defines the desired state of Web
type WebSpec struct {
	// INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
	// Important: Run "make" to regenerate code after modifying this file

	Type string  `json:"type"`
	Spec v1.JSON `json:"spec"`
}
```

这里增加两个字段，一个是tye，一个是spec。其中type指定这个CRD实际依赖的子CRD的Kind。当然，demo里面就只有Blog了。spec字段类型设置成了JSON，用于存储type需要的参数。这里之前尝试直接使用`map[string]interface{}`，但是`kubebuilder`会无法生成deepcopy。用v1.JSON类型，可以直接获取到yaml里面spec字段下面的整个json信息。

这里非常简化，不修改status什么的。

下面是修改controller：
```go
ctx := context.Background()
log := r.Log.WithValues("web", req.NamespacedName)

// your logic here
web := webv1.Web{}
err := r.Get(ctx, req.NamespacedName, &web)

...

webType := web.Spec.Type
groupVersionKind := web.GroupVersionKind()

log.Info("get web success", "type", webType, "gvk", groupVersionKind)
```

这里同样忽略异常处理，首先获取对应的web CR，获取其中配置的type和spec。

然后需要查询对应type的CR是否已经创建：
```go
cr := unstructured.Unstructured{}
cr.SetGroupVersionKind(schema.GroupVersionKind{
    Group:   groupVersionKind.Group,
    Version: groupVersionKind.Version,
    Kind:    webType,
})
err = r.Get(ctx, req.NamespacedName, &cr)
```

如果没有找到对应的CR，则创建一个新的：
```go
if err != nil && errors.IsNotFound(err) {
    log.Info("associate cr not found", "type", webType)
    newCr, err := r.newCr(&web)
    ...
    err = r.Create(ctx, newCr)
    ...

    log.Info("create cr success", "type", webType)
}
```

具体创建CR的函数：
```go
func (r *WebReconciler) newCr(web *webv1.Web) (*unstructured.Unstructured, error) {
	cr := &unstructured.Unstructured{
		Object: map[string]interface{}{
			"spec": web.Spec.Spec,
		},
	}
	gvk := web.GroupVersionKind()
	cr.SetGroupVersionKind(schema.GroupVersionKind{
		Group:   gvk.Group,
		Version: gvk.Version,
		Kind:    web.Spec.Type,
	})
	cr.SetName(web.Name + "-" + strings.ToLower(web.Spec.Type))
	cr.SetNamespace(web.Namespace)

	err := controllerutil.SetControllerReference(web, cr, r.Scheme)
	if err != nil {
		return nil, err
	} else {
		return cr, nil
	}
}
```
这里先初始化一个Unstructured，并设置spec为web CR中设置的spec。然后设置CR的其他属性，包括GVK坐标和名称，名称这里需要转下小写，否则创建的时候会报错，提示DNS名称不支持。

完成编码之后，修改下生成的sample（config/samples/web_v1_web.yaml）：
```yaml
apiVersion: web.helloworld/v1
kind: Web
metadata:
  name: web-sample
spec:
  # Add fields here
  type: Blog
  spec:
    url: http://a.com
```

添加一些必须的字段，包括type和spec，然后提交给k8s：
```
kubectl apply -f config/samples/web_v1_web.yaml
```

这时候控制台输出：
```
2020-07-10T11:37:09.393+0800	INFO	controllers.Web	get web success	{"web": "default/web-sample", "type": "Blog", "gvk": "web.helloworld/v1, Kind=Web"}
2020-07-10T11:37:09.401+0800	INFO	controllers.Web	associate cr not found	{"web": "default/web-sample", "type": "Blog"}
2020-07-10T11:37:09.408+0800	INFO	controllers.Web	create cr success	{"web": "default/web-sample", "type": "Blog"}
2020-07-10T11:37:09.408+0800	DEBUG	controller-runtime.controller	Successfully Reconciled	{"controller": "web", "request": "default/web-sample"}
2020-07-10T11:37:09.408+0800	INFO	controllers.Blog	get blog success	{"blog": "default/web-sample-blog", "url": "http://a.com"}
2020-07-10T11:37:09.509+0800	INFO	controllers.Blog	deployment not found, start to create	{"blog": "default/web-sample-blog"}
2020-07-10T11:37:09.523+0800	INFO	controllers.Blog	create deployment success	{"blog": "default/web-sample-blog"}
2020-07-10T11:37:09.626+0800	INFO	controllers.Blog	service not found, start to create	{"blog": "default/web-sample-blog"}
2020-07-10T11:37:09.640+0800	INFO	controllers.Blog	create service success	{"blog": "default/web-sample-blog"}
2020-07-10T11:37:09.640+0800	DEBUG	controller-runtime.controller	Successfully Reconciled	{"controller": "blog", "request": "default/web-sample-blog"}
```

从日志可以看见，web的controller创建了一个CR，被blog的controller监听并处理了。

再创建一个：
```
cat config/samples/web_v1_web.yaml | sed 's/web-sample/web-sample1/' | kubectl apply -f -
```

然后通过`kubectl`去k8s上查看下真正的资源创建情况：

```
#kubectl get pod
NAME                                READY   STATUS    RESTARTS   AGE
web-sample-blog-67c86ddcb5-mp4gq    1/1     Running   0          3m12s
web-sample1-blog-76646dcf4f-lmhqc   1/1     Running   0          2m39s
```

可以看见创建了2个pod。

```
kubectl get service
NAME                       TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
web-sample-blog-service    LoadBalancer   10.98.65.16     <pending>     8080:31658/TCP   10m
web-sample1-blog-service   LoadBalancer   10.105.16.163   <pending>     8080:30944/TCP   10m
```
同时创建了2个LB。

这样，我们就能够通过多个CRD，将功能拆开了。
