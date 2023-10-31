
# 网络编程包java.net
## 自动获取服务器的ip地址
两种方式， 方式1:使用InetAddress类获取本机IP地址，方式2:使用NetworkInterface类获取本机IP地址

方式1获取linux本机地址时InetAddress.getLocalHost() 方法通常会返回主机名和一个包含 127.0.0.1 的回送地址。
如果主机名无法解析，则该方法将返回回送地址而不是实际的 IP 地址。因此，使用 NetworkInterface 类获取ip地址比较好。

``` java
//式1:使用InetAddress类获取本机IP地址
InetAddress localAddress = InetAddress.getLocalHost();
String ipAddress = localAddress.getHostAddress();
//方式2:使用NetworkInterface类获取本机IP地址
Enumeration<NetworkInterface> interfaces = NetworkInterface.getNetworkInterfaces();
while (interfaces.hasMoreElements()) {
    NetworkInterface networkInterface = interfaces.nextElement();
    if (networkInterface.isUp() && !networkInterface.isLoopback()) {
        Enumeration<InetAddress> addresses = networkInterface.getInetAddresses();
        while (addresses.hasMoreElements()) {
            InetAddress address = addresses.nextElement();
            if (address instanceof Inet4Address) {
                String ipAddress = address.getHostAddress();
                // do something with ipAddress
            }
        }
    }
}
```

## 检测端口是否被占用
``` java

```