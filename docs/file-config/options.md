# 服务器参数

服务器参数主要为运行时无法动态变更的部分参数，如有变更最好重启实例

主要包含以下内容

- [ReverseProxyOptions](/VKProxy.Doc/docs/file-config/ReverseProxyOptions) 路由相关参数
- [ServerOptions](/VKProxy.Doc/docs/file-config/ServerOptions)  服务功能配置
- [SocketTransportOptions](/VKProxy.Doc/docs/file-config/SocketTransportOptions)  Kestrel内部对于tcp处理的配置 
- [UdpSocketTransportOptions](/VKProxy.Doc/docs/file-config/UdpSocketTransportOptions)  简单实现的 udp 支持对应配置
- [QuicTransportOptions](/VKProxy.Doc/docs/file-config/QuicTransportOptions)  Kestrel内部对于Quic处理的配置 