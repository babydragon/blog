# 从0开始学kubernetes——用rust写个CRD


k8s带动了golang的生态，但是k8s本身的扩展机制是通过http接口实现的。因此，扩展的API有基于[各种语言](https://kubernetes.io/zh/docs/reference/using-api/client-libraries/)的。虽然rust客户端没有官方支持，但也有了基本可用的库。这次就用`rust-rs`这个库做一个初步尝试。

首先是添加依赖：

```toml
[dependencies]
kube = "0.35.1"
kube-derive = "0.35.1"
k8s-openapi = { version = "0.8.0", default-features = false, features = ["v1_18"] }
serde = "1.0"
serde_json = "1.0"
log = "0.4"
env_logger = "0.7"
tokio = { version = "0.2", features = ["macros", "rt-core"] }
anyhow = "1.0"
```

官方的example里面没有具体说明详细的依赖，这里注意`tokio`依赖了异步，同时用了异步main函数，因此需要增加两个feature。

完整代码很短，这里整个贴出来：
```rust
#[macro_use]
extern crate log;

use kube_derive::CustomResource;
use serde::{Serialize, Deserialize};
use kube::{Client, Api};
use kube::api::{ListParams, Meta};
use kube::runtime::Reflector;

// 1
#[derive(CustomResource, Deserialize, Serialize, Clone, Debug)]
#[kube(group = "web.helloworld", version = "v1", namespaced)]
pub struct BlogSpec {
    url: String
}

// 2
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // 3
    std::env::set_var("RUST_LOG", "info,kube=debug");
    env_logger::init();

    // 4
    info!("start to connect k8s cluster");
    let client = Client::try_default().await?;
    let ns = std::env::var("NAMESPACE").unwrap_or("default".into());

    // 5
    let api: Api<Blog> = Api::namespaced(client, &ns);

    // 6
    let params = ListParams::default();
    let reflector = Reflector::new(api).params(params);

    let rf_clone = reflector.clone();

    // 7
    tokio::spawn(async move {
        loop {
            tokio::time::delay_for(std::time::Duration::from_secs(5)).await;
            info!("start to read cr list");
            let crds = rf_clone.state().await.unwrap();

            for crd in crds {
                info!("cr name: {}", Meta::name(&crd))
            }
        }
    });

    // 8
    reflector.run().await?;
    Ok(())
}
```

大致的流程：
1. 定义个CRD的结构体，这里用了`kube-derive`crate里面的注解，因此只需要写一个spec对象即可。为了直接复用前文用`kubebuilder`生成的sample，这里定义完全相同的Kind和spec。实际这个derive会展开生成`Blog`结构体。
2. 这里使用rust的异步化语义，因此main函数也定义成了async
3. 初始化日志需要用到的环境变量
4. 初始化client，这里用了默认的方式。首先会通过环境变量判断是否在k8s集群内，如果不存在，会继续读取`kubectl`的配置文件。配置文件会尝试读取`KUBECONFIG`环境变量，读取配置文件。如果环境变量不存在，会读取默认路径（`$HOME/.kube/config`）中的配置文件。这里有个坑。
5. 初始化API对象
6. 初始化`reflector`对象。`kube-rs`中定义的reflector是一个会周期性拉取指定对象，并缓存
7. 新建一个异步任务，定时读取`reflector`中的state，打印其中缓存的cr名字
8. 开始`reflector`拉取

运行之前，需要先提交下CRD，因为声明的和前文相同，直接用之前的即可。

然后执行：
```
[2020-07-17T06:53:44Z INFO  crd_demo] start to connect k8s cluster
[2020-07-17T06:53:44Z DEBUG kube::runtime::reflector] Initialized with: [blog-sample [default]]
[2020-07-17T06:53:49Z INFO  crd_demo] start to read cr list
[2020-07-17T06:53:49Z INFO  crd_demo] cr name: blog-sample
[2020-07-17T06:53:54Z INFO  crd_demo] start to read cr list
[2020-07-17T06:53:54Z INFO  crd_demo] cr name: blog-sample
```

这里可以看见，周期性读取到了缓存的cr数据。

使用`kubectl`删除这个cr之后，输出：
```
[2020-07-17T06:54:10Z DEBUG kube::runtime::reflector] Removing blog-sample from Blog
[2020-07-17T06:54:14Z INFO  crd_demo] start to read cr list
[2020-07-17T06:54:19Z INFO  crd_demo] start to read cr list
```
可以看见删除之后，再读取列表为空。

最后退出：
```
[2020-07-17T06:54:23Z INFO  kube::runtime::reflector] Received ctrl_c, exiting
```
因为`reflector`会监听TERM信号，这里直接退出能够监听到，并正常退出。

## 坑

毕竟不是官方支持的库，使用还是有些坑的：

1. 本地`KUBECONFIG`环境变量，如果配置了多个文件（实际`kubectl`也是推荐这种方式来增加集群配置的），会无法解析，提示文件不存在。实际应该按照分隔符分隔下，才能正常解析。目前运行的时候在clion中重新覆盖了`KUBECONFIG`环境变量，只指定了minikube那个，才正常初始化。
2. 还没有像controller-runtime那样有很好的controller封装。目前上层封装就只有`Informer`和`Reflector`，前者需要自己去关注poll事件，后者poll的事件只是简单的缓存，无法及时处理。看`kube-rs`的[issue](https://github.com/clux/kube-rs/issues/148)和分支，已经有controller实现在进行，但是看上去进度不大，版本和当前版本已经有差距了。