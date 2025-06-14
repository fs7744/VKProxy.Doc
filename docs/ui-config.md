# 通过UI站点配置

由于文件配置存在一定使用难度，所以也有提供简单的 ui配置站点[VKProxy.Web](https://github.com/fs7744/VKProxy.Web)

>[!WARNING]
>由于文件分发会导致大家部署多实例的难度，所以 ui 站点目前只支持 etcd 作为配置源， 同时服务器参数相关无法通过ui站点配置, 请使用文件会程序配置 参见[服务器参数](/VKProxy.Doc/docs/file-config/options)


首先启动一个 etcd （可参考 [Run etcd clusters inside containers](https://etcd.io/docs/v3.4/op-guide/container/))

``` shell
export NODE1=127.0.0.1

ETCD_VERSION=v3.4.37
REGISTRY=quay.io/coreos/etcd
# available from v3.2.5
REGISTRY=gcr.io/etcd-development/etcd

docker run \
  -p 2379:2379 \
  -p 2380:2380 \
  --volume=${DATA_DIR}:/etcd-data \
  --name etcd ${REGISTRY}:${ETCD_VERSION} \
  /usr/local/bin/etcd \
  --data-dir=/etcd-data --name node1 \
  --initial-advertise-peer-urls http://${NODE1}:2380 --listen-peer-urls http://0.0.0.0:2380 \
  --advertise-client-urls http://${NODE1}:2379 --listen-client-urls http://0.0.0.0:2379 \
  --initial-cluster node1=http://${NODE1}:2380
```

VKProxy agent 启动参考 [安装](/VKProxy.Doc/docs/install)

UI docker 部署

参数可以使用如下

- ETCD_CONNECTION_STRING

  etcd address, like http://127.0.0.1:2379

  example `ETCD_CONNECTION_STRING=http://127.0.0.1:2379`

- ETCD_PREFIX

  default is /ReverseProxy/

  example `ETCD_PREFIX=/ReverseProxy/`

- ASPNETCORE_URLS

  example `ASPNETCORE_URLS=http://*:80`

举例：

``` shell
docker run --rm -e ETCD_CONNECTION_STRING=http://127.0.0.1:2379 -e ASPNETCORE_URLS=http://*:8770 --network host vkproxy/ui:0.0.0.7

// 启动后会看到类似输出
warn: Microsoft.AspNetCore.Hosting.Diagnostics[15]
      Overriding HTTP_PORTS '8080' and HTTPS_PORTS ''. Binding to values defined by URLS instead 'http://*:8770'.
info: Microsoft.Hosting.Lifetime[14]
      Now listening on: http://[::]:8770
info: Microsoft.Hosting.Lifetime[0]
      Application started. Press Ctrl+C to shut down.
info: Microsoft.Hosting.Lifetime[0]
      Hosting environment: Production
info: Microsoft.Hosting.Lifetime[0]
      Content root path: /app
```

然后你就可以在浏览器 访问 http://127.0.0.1:8770 使用 UI 了

![webui.jpg](/VKProxy.Doc/images/webui.jpg)

具体使用可参考


- [不同监听场景如何配置](/VKProxy.Doc/docs/howtolisten)
- [如何为HTTP配置路由复杂匹配](/VKProxy.Doc/docs/statement)
- [如何为HTTP配置请求和响应转换](/VKProxy.Doc/docs/transforms)