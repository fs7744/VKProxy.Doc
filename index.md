# ����
[VKProxy](https://github.com/fs7744/VKProxy) ��ʹ��c#�����Ļ��� [Kestrel](https://github.com/dotnet/aspnetcore/tree/main/src/Servers/Kestrel) ʵ�� L4/L7�Ĵ���

## Ϊ�λ��� Kestrel

��Ҫ��http Э��Ĵ���ʵ��̫���ˣ��������޴�����Ϊ��ʡ�¾ͻ��� Kestrel ��

����������֪ Kestrel �� Aspnetcore Ϊ�˿�ƽ̨��ʵ�ֵ�web server��ֻ�ṩ http 1/2/3 �� L7�������

������Դ���ͬѧ��֪������ʵ�䱾���L4��(socket)ʵ�ֵ�HttpЭ�鴦��ֻ��[OnBind](https://github.com/dotnet/aspnetcore/blob/main/src/Servers/Kestrel/Core/src/Internal/KestrelServerImpl.cs#L137)ֻ��http���ʵ���Լ�û���ṩ��ع�����չ��api������������������

���Ǽ�Ȼ�����ǿ�Դ�ģ���������Ҳ֪��dotnet����Ȼ�鷳�����ܿ�Խ�������Ƶ�������Reflection�����������ǲ����赲���ǵ�ħצ

��ps 
     1. ���������ƹ����ƿ��ܻ���[Native AOT](https://learn.microsoft.com/zh-cn/dotnet/core/deploying/native-aot/?tabs=windows%2Cnet8)��س����������⣬Ŀǰ��ʱû����������ز���
     2. �ڲ�ͬ�汾Kestrel ���ܻ����api�䶯��ĿǰΪ��ʡ�£���������汾���죬��ʱ��net9.0Ϊ׼��net10��ʽ������Ǩ��������net10���˺�������net9.0֮ǰ�汾
 ��

### ����

���ò�����һ�����ޣ�dotnet ��socket û���ṩͳһ�Ŀ����socketת��api����Ϊdotnet�ǿ�ƽ̨�ģ���ͬϵͳ���ڲ��죬��issue [Migrate Socket between processes](https://github.com/dotnet/runtime/issues/48637) �Ѿ�����û��������

���Բ�����������������ʱ����֧��

�����ڲ���֧�ּ������ñ䶯��������ؼ����˿ڱ䶯����ȣ����Դ󲿷ֳ���Ӧ��û��̫�����⣬ֻ���޷�����tcp����Ǩ��

# ����


- [X] TCP proxy
- [X] UDP proxy
- [X] HTTP1/2/3 proxy
- [X] SNI proxy (no tls handle, tls base on upstream)
- [X] SNI proxy (tls handle, upstream no tls handle)
- [X] dns (use system dns, no query from dns server )
- [X] LoadBalancingPolicy
- [X] Passive HealthCheck
- [X] TCP Connected Active HealthCheck
- [X] Configuration 
- [X] reload config and rebind
- [X] socks5 TCP
	- [X] NO AUTH
	- [X] simple user/password auth
- [X] socks5 UDP
- [X] Http Active HealthCheck
- [X] socks5(tcp) to websocket to socks5