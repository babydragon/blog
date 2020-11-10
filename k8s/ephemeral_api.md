# 使用API操作临时容器

## 背景
之前参照k8s官方文档使用`kubectl`操作了临时容器，但真实业务场景不可能直接使用命令行工具，还是需要API进行接入。

但是由于临时容器仍然处于alpha阶段，client-go一直都没有专门封装，因此只能通过最基本的raw api和k8s进行交互。

## 准备
临时容器作为alpha阶段的feature，本地验证一定要注意开启feature gate。对于minikube，可以直接加上配置：

```
minikube config set feature-gates EphemeralContainers=true
```

避免每次启动的时候遗忘。

## 实现

k8s api中，已经包含了`EphemeralContainer`这个类型，因此可以直接通过该类型创建完整的临时容器描述：

```
func newEphemeralContainer(pod *corev1.Pod) (*corev1.EphemeralContainer, error) {
	// 这里添加业务逻辑

	return &corev1.EphemeralContainer{
		EphemeralContainerCommon: corev1.EphemeralContainerCommon{
			Name:      option.ephemeralContainerName,       // 临时容器名称，pod中需要唯一，且目前不能删除，需要有唯一生成规则
			Image:     img,                                 // 容器使用的镜像
			Command:   []string{javaDumpCommand},
			Args:      args,
			Resources: corev1.ResourceRequirements{},
			VolumeMounts: []corev1.VolumeMount{*mount},
			ImagePullPolicy:          "Always",
			TerminationMessagePolicy: "FallbackToLogsOnError",
			TTY:                      false,
			Stdin:                    false,
			Env:                      env,
		},
		TargetContainerName: targetContainer,               // 目标容器，从1.18版本开始，临时容器会加入目标容器ns，无需开启pod层面共享
	}, nil
}
```

创建出临时容器对象之后，需要通过dynamic client来向k8s apiserver发起PUT请求。

```
func updateEphemeralContainer(pod *corev1.Pod, ephemeralContainers *corev1.EphemeralContainers) error {
	// FIXME: 这里需要用已经创建出来的client
	config, err := ctrl.GetConfig()

	if err != nil {
		logging.Default.Error(err, "fail to get config")

		return err
	}

	c, err := dynamic.NewForConfig(config)
	if err != nil {
		logging.Default.Error(err, "fail to get rest client")

		return err
	}

	gvk := pod.GroupVersionKind()
	dynamicClient := c.Resource(schema.GroupVersionResource{
		Group:    gvk.Group,
		Version:  gvk.Version,
		Resource: "pods",
	}).Namespace(pod.Namespace)

    // 由于只能使用raw api，这里需要转成unstructured数据结构
	unstructuredData, err := runtime.DefaultUnstructuredConverter.ToUnstructured(ephemeralContainers)
	if err != nil {
		logging.Default.Error(err, "fail to convert to unstructured")

		return err
	}

    // 使用dynamicClient更新pod中的ephemeralcontainers子资源，update操作实际会发起PUT请求
	_, err = dynamicClient.Update(&unstructured.Unstructured{Object: unstructuredData},
		metav1.UpdateOptions{}, "ephemeralcontainers")
	if err != nil {
		logging.Default.Error(err, "fail to update ephemeral container")

		return err
	}

	return nil
}
```

当然由于之前工程结构的关系每次创建了client，这个是没有必要的。这里需要特别注意的几点：

1. 临时容器目前不能删除，pod级别name必须唯一，因此一定要有一个单pod不会重复的名字生成规则；
2. 目前client-go不支持直接更改`ephemeralContainers`子资源，因此需要转换成unstructured对象，同时在更新时指定`ephemeralContainers`子资源。这里和使用`kubetl --raw replace`实际发起的请求是相同的，client-go的dynamicClient会自动从unstructured对象中提取对象url，并进行更新。

## 目前k8s对临时容器的支持
由于临时容器只是一个alpha级别的feature，默认也是关闭的，目前官方的[描述](https://github.com/kubernetes/enhancements/issues/277)是：

* [x] Ephemeral containers added to core API (likely 1.16)
* [x] kubelet support for creating basic ephemeral containers (likely 1.16)
* [x] kubectl command to launch ephemeral containers (likely 1.17)
* [x] kubelet support for namespace targeting (possibly 1.18)
* [x] kubectl support for adding ephemeral containers (likely 1.18)
* [ ] allow setting securityContext (target 1.20)
* [ ] allow removing ephemeralContainers (target 1.20)

期待20版本能够支持移除功能吧。