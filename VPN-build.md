

架构: 本地 ---vpn----> ucloud上的download主机 ----> 访问(ucloud上的deploy上的gitlab和jenkins)和其他任意非download主机的地址


1.拉取vpn镜像:
docker pull kylemanna/openvpn:2.4

2.创建vpn配置文件在本机的存放目录
mkdir -p /data/openvpn

3.生成配置文件
docker run -v /data/openvpn:/etc/openvpn --rm kylemanna/openvpn:2.4 ovpn_genconfig -u tcp://1.1.1.1

4.生成密钥文件(密码admin123456)
docker run -v /data/openvpn:/etc/openvpn --rm -it kylemanna/openvpn:2.4 ovpn_initpki
`````
输入私钥密码（输入时是看不见的）：
Enter PEM pass phrase:admin123456
再输入一遍
Verifying - Enter PEM pass phrase:admin123456
输入一个CA名称（我这里直接回车）
Common Name (eg: your user, host, or server name) [Easy-RSA CA]:
输入刚才设置的私钥密码（输入完成后会再让输入一次）
Enter pass phrase for /etc/openvpn/pki/private/ca.key:admin123456
`````

5.生成客户端证书(zhangsan替换成用户名)
docker run -v /data/openvpn:/etc/openvpn --rm -it kylemanna/openvpn:2.4 easyrsa build-client-full zhangsan nopass

6.导出客户端证书
mkdir -p /data/openvpn/conf
docker run -v /data/openvpn:/etc/openvpn --rm kylemanna/openvpn:2.4 ovpn_getclient zhangsan > /data/openvpn/conf/zhangsan.ovpn

10.启动VPN (注意选择连接类型为TCP)
docker run --name openvpn -v /data/openvpn:/etc/openvpn -d -p 1194:1194/tcp --cap-add=NET_ADMIN kylemanna/openvpn:2.4

11.设置防火墙规则(略)

12.下载客户端证书给客户使用
sz /data/openvpn/conf/zhangsan.ovpn


13. mac用户下载Tunnelblick,windows用户下载openvpn客户端安装该证书



13.为了方便管理: 使用如下两个脚本来删除用户和创建用户
##创建用户
-----------------------
```
#!/bin/bash
read -p "please your username: " NAME
docker run -v /data/openvpn:/etc/openvpn --rm -it kylemanna/openvpn:2.4 easyrsa build-client-full $NAME nopass
docker run -v /data/openvpn:/etc/openvpn --rm kylemanna/openvpn:2.4 ovpn_getclient $NAME > /data/openvpn/conf/"$NAME".ovpn
docker restart openvpn
注意: 需要手动修改xx.ovpn文件中的vpn端口为11194 (默认是1194) --- 否则不能连接
```
-----------------------
## 删除用户
-----------------------
```
#!/bin/bash
read -p "Delete username: " DNAME
docker run -v /data/openvpn:/etc/openvpn --rm -it kylemanna/openvpn:2.4 easyrsa revoke $DNAME
docker run -v /data/openvpn:/etc/openvpn --rm -it kylemanna/openvpn:2.4 easyrsa gen-crl
docker run -v /data/openvpn:/etc/openvpn --rm -it kylemanna/openvpn:2.4 rm -f /etc/openvpn/pki/reqs/"$DNAME".req
docker run -v /data/openvpn:/etc/openvpn --rm -it kylemanna/openvpn:2.4 rm -f /etc/openvpn/pki/private/"$DNAME".key
docker run -v /data/openvpn:/etc/openvpn --rm -it kylemanna/openvpn:2.4 rm -f /etc/openvpn/pki/issued/"$DNAME".crt
docker restart openvpn
rm -rf  /data/openvpn/conf/$DNAME.ovpn
```
-----------------------
注意: 删除一个用户后,需要删除其在/data/openvpn/conf/下的username.ovpn文件




