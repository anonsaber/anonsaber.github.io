很多时候需要在 Python 获取指定网卡的 IP 地址。如果能够在目标机器上安装 netifaces 模块，事情会变得非常容易，如果安装模块是不方便的话，使用 Python 的标准库同样也能实现。这里分别记录一下两种办法。

## 使用模块实现

```python
import netifaces

interface_list = netifaces.interfaces()
interface_name = list(filter(lambda x: x.startswith('e'), interface_list))[0]
ipaddr = netifaces.ifaddresses(interface_name)[netifaces.AF_INET][0]['addr']
```

## 使用标准库实现

```python
import os
import socket
import fcntl
import struct

interface_list = os.listdir('/sys/class/net/')
interface_name = list(filter(lambda x: x.startswith('e'), interface_list))[0]

def get_ip_address(ifname):
    s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    try:
        ip = socket.inet_ntoa(fcntl.ioctl(
            s.fileno(),
            0x8915,
            struct.pack('256s', ifname[:15].encode('utf-8')))[20:24])
    except OSError:
        print("Wrong interface name: " + ifname + "ncheck config.ini")
        sys.exit(0)
    return ip
```
<!-- ##{"timestamp":1620662400}## -->