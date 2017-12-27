# 使用Ubuntu 16.04搭建OpenVPN服务器

##1. 安装openvpn与easy-rsa

```
$ sudo apt update && sudo apt install openvpn easy-rsa -y
```

##2. 构建CA

OpenVPN采用TLS/SSL协议进行加密，所以我们需要使用CA来签署服务端和客户端证书。首先我们使用easy-rsa的模板来设置一个简单的CA

1. 复制easy-rsa的模板目录到～/openvpn-ca

```
$ make-cadir ~/openvpn-ca
$ cd ~/openvpn-ca
```

2. 配置CA环境变量
```
$ vim vars
```
然后我们找到如下的一段，自己进行修改，注意最后一句改为server
```
# These are the default values for fields
# which will be placed in the certificate.
# Don't leave any of these fields blank.
export KEY_COUNTRY="CN"
export KEY_PROVINCE="Hubei"
export KEY_CITY="Wuhan"
export KEY_ORG="whu"
export KEY_EMAIL="×××××××××@whu.edu.cn"
export KEY_OU="openvpn"

#X509 Subject Field
export KEY_NAME="server"
```
使用如下命令让环境变量生效
```
$ source vars
```
构建CA，回车就好
```
$ ./clean-all
$ ./build-ca
```
##3. 生成服务端证书，密钥

首先生成服务器证书与Key，回车就好
```
$ ./build-key-server server
```
然后生成DH交换Key，可能需要较长时间
```
$ ./build-dh
```
生成HMAC签名加强TLS的认证
```
$ openvpn --genkey --secret keys/ta.key
```

##4. 生成客户端证书，密钥

我们直接在服务器端生成客户端的证书与密钥，然后直接使用CA签名。首先为一个客户端生成证书，如果有多个客户端就生成多个证书
```
$ ./build-key client1
```
##5. 配置OpenVPN服务

把服务器证书、key；CA证书、Key复制到/etc/openvpn目录下
```
$ cd ./keys
$ sudo cp ca.crt ca.key server.crt server.key ta.key dh2048.pem /etc/openvpn
```
使用模板创建OpenVPN server的配置文件
```
$ gunzip -c /usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz | sudo tee /etc/openvpn/server.conf
```
然后编辑模板配置文件
```
$ sudo vim /etc/openvpn/server.conf
```
去掉如下的注释
```
tls-auth ta.key 0 # This file is secret
key-direction 0
```

##6. 打开IP转发
```
$ sudo vim /etc/sysctl.conf
```
打开如下注释
```
net.ipv4.ip_forward=1
```
执行命令使生效
```
$ sudo sysctl -p
```
##7. 启动OpenVPN服务
```
$ sudo systemctl start openvpn@server
$ sudo systemctl enable openvpn@server
```
##8. 生成OpenVPN客户端使用的 .ovpn配置文件
创建目录并复制模板的配置文件
```
$ mkdir -p ~/client/ovpn
$ cp /usr/share/doc/openvpn/examples/sample-config-files/client.conf ~/client/
```
编辑配置文件并作出如下编辑如下
```
$ vim ~/client/client.conf
```

```
# 修改IP
remote my-server-1  1194
...
# 去掉注释
user nobody
group nogroup
...
# 修改为inline
ca [inline]
cert [inline]
key [inline]
...
#添加如下
tls-auth [inline] 1
key-direction 1
```
生成ovpn文件
```
$ cat ~/client/client.conf <(echo -e '<ca>') ~/openvpn-ca/keys/ca.crt <(echo -e '</ca>\n<cert>') ~/openvpn-ca/keys/client1.crt <(echo -e '</cert>\n<key>') ~/openvpn-ca/keys/client1.key <(echo -e '</key>\n<tls-auth>') ~/openvpn-ca/keys/ta.key <(echo -e '</tls-auth>') > ~/client/ovpn/client1.ovpn
```
把client1.ovpn传给客户端就能使用该配置文件连接到VPN