这些工具正常情况下只能工作于 linux/mac 系统之上，如果你使用的是 Windows，以`.sh`结尾的脚本应该无法正常工作。

> 使用之前，请确保已安装 adb 以及启动设备上的 lamda 服务端。
> 需要注意，一次只能连接一个设备到当前电脑。所以脚本均不支持同时插入的多个USB ADB设备。

## 前置条件
安装所需的依赖包
```bash
pip3 install -r requirements.txt
```

为了确保部分通过本地USB操作的功能正常进行，请在开始前连接USB线并开启授权**开发者调试**功能，随后执行
```bash
adb forward tcp:65000 tcp:65000
```

如果你在服务端启用了通信加密(certificate)，你需要在使用前设置环境变量 `CERTIFICATE`
```bash
export CERTIFICATE=/path/to/lamda.pem
```

192.168.1.2 为示例IP，请自行获取设备的实际IP地址。

## cert.sh

用来生成用于传输加密的证书
```bash
bash cert.sh
```

如果你需要生成特定 CN 的证书
```bash
bash cert.sh device1.example.com
```

即可，当前目录下的 lamda.pem 或者 device1.example.com.pem 即为所需的证书。
在服务端应用该证书后，客户端也必须使用此证书来进行远程调用以及 ssh 连接。
应用此证书后，将会加密任何通过 client 库进行的操作，ssh 将不再使用默认的秘钥。
同时，web 控制页面将需要登录（你可以在证书最后一行找到此登录密码），

如果操作系统不方便使用此工具，当前目录下已经为你准备好了一个默认的 SSL 证书

```bash
test.id_rsa # 用于连接 SSH 的对应 ID_RSA
test.pem    # 用于加密客户端与服务端通信的证书
```

## startmitm.py

启动中间人，这将会完全自动的在设备上开启全局的中间人，你就可以截获应用的 http/s 流量，当然，也包括 DNS 请求。

首先确保当前电脑与设备在同一个网段，192.168.1.2 为运行了 lamda 的手机设备。
其次，确保你已在命令行验证 mitmproxy 已安装成功（在命令行输入 `mitmdump` 进行验证）。
如果你是首次使用 mitmproxy，请在执行脚本前手动执行一次命令 `mitmdump` 并退出来完成
mitmproxy 的初始化操作。

执行
```bash
python3 -u startmitm.py 192.168.1.2
```
即可。

如果需要截获特定应用的流量而不是全局
```bash
python3 -u startmitm.py 192.168.1.2:com.some.package
```
即可。

如果需要传递其他参数到 mitmproxy，例如 -s，则执行
```bash
python3 -u startmitm.py 192.168.1.2 -s http_flow_hook.py
```
任何 192.168.1.2 之后的参数将传递给 mitmproxy

即使手机与当前电脑不在同一网段，你仍然可以使用在本机截获流量，但是需要先完成**前置条件**。
```bash
python3 -u startmitm.py localhost
```
即可。

注意：你可能需要完全结束APP并重新打开使其生效。

按下一次 `CONTROL` + `C` 退出脚本。

### DNS 中间人

截获 DNS 请求需要确保 mitmproxy 的版本 >= 8.1.0，且需要以**管理员**或者**root**身份运行脚本。
```bash
python3 -u startmitm.py 192.168.1.2 --set dns_server=true
```
即可。

这些DNS请求默认会从本机发出，你也可以将这些 DNS 请求转发到指定的上游DNS服务器例如 `1.1.1.1`。
```bash
python3 -u startmitm.py 192.168.1.2 --set dns_server=true --set dns_mode=reverse:1.1.1.1
```

hook 脚本的方法名称定义有一些变化，正常 http 请求为 `response()`，截获 DNS 时需要使用 `dns_response()`。

```python
def response(flow):
    print (flow, type(flow))

def dns_response(flow):
    print (flow, type(flow))
```

具体请查看 mitmproxy 的文档。

## ssh.sh

连接入手机上的 shell 终端。

执行
```bash
bash ssh.sh 192.168.1.2
```
即可。

同样，对于不同网段但通过USB连接的设备，也可以
```bash
bash ssh.sh localhost
```

## scp.sh

使用 `scp` 复制设备上的文件到本地。

将 192.168.1.2 的 `/sdcard/DCIM` 目录复制到当前目录

```bash
bash scp.sh 192.168.1.2:/sdcard/DCIM .
```

将本地目录/文件 `test/` 复制到设备 192.168.1.2 的 `/sdcard/`

```bash
bash scp.sh test/ 192.168.1.2:/sdcard
```

## discover.py

列出本地网络中所有在线的设备（部分网络情况下可能一些设备没有列出，请多次尝试）

```bash
python3 discover.py
```

## emu-install.sh

仅适用于主流模拟器的服务端安装脚本，需要设备开启 WIFI adb，需要提前下载
对应架构服务端安装包到当前运行目录。

```bash
bash emu-install 192.168.1.2
```

## globalmitm/

用于分析需要使用代理才能访问的应用的流量，是一个整合的 docker 镜像，需要支持 UDP 协议的 shadowsocks 服务，请自行获取该服务。
因为 Windows/Mac 上的 docker 实现问题，此镜像必须配合 `Linux` 系列系统使用。

镜像已经构建并上传至 dockerhub，你也可以自行构建。

首先拼接代理为以下链接格式

```bash
ss://加密算法:密码@地址:端口
```

现在执行命令

```bash
docker run --rm -it --net host -e SS=ss://加密算法:密码@地址:端口 rev1si0n/mitm 192.168.x.x
```
