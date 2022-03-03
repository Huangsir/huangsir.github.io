---
layout: post
title: "iphone用ikev2搭梯子"
date: 2015-10-24 21:38:19 +8000
categories: []
---
<!-- datetime: 2015-10-24 21:38:19 -->

一种个人感觉比较小众的科学Shangwang的方式

<!-- more -->



想到要为我的iphone翻墙原因有两个。一是前几天出门旅游，在玩instagram；二是我一直在玩ingress，但是手头iphone翻墙实在不方便，虽然在家大部分时间还是用Android机开画图工具-，-#

不过始终，觉得为自己的iphone弄个翻墙还是很有必要的。

为此我尝试了使用openvpn。但发现openvpn在如果是连接国内ip的server是十分稳定的，但是如果连接国外的openvpn服务，就会经常被block掉连接。

这样我很不爽，出门打架，或者面基的时候被block了，还要花半天时间来连接服务器，这尼玛不得坑死我。

于是我就发现了ikev2这套协议。



选用的方法是在在vps上部署`strongswan`，部署的方法可以见[这篇文章](https://gist.github.com/losisli/11081793)

然而相比于大部分blog里直接wget下来安装的做法，本人喜欢使用apt-get来安装。

---

**安装strongswan**

``` bash
apt-get install build-essential     #编译环境
aptitude install libgmp10 libgmp3-dev libssl-dev pkg-config libpcsclite-dev libpam0g-dev     #编译所需要的软件
apt-get install strongswan
```

---

**生成证书**

生成根证书

``` bash
ipsec pki --gen --outform pem > caKey.pem
ipsec pki --self --in caKey.pem --dn "C=CN, O=myNetwork, CN=myCA" --ca --outform pem > caCert.pem
```

生成服务器证书

``` bash
ipsec pki --gen --outform pem > serverKey.pem
ipsec pki --pub --in serverKey.pem | ipsec pki --issue --cacert caCert.pem --cakey caKey.pem --dn "C=CN, O=myNetwork, CN=<这里要填写服务器的ip或者域名>" --san="<这里要填写服务器的ip或者域名>" --flag serverAuth --flag ikeIntermediate --outform pem > serverCert.pem
```

生成客户端证书

``` bash
ipsec pki --gen --outform pem > clientKey.pem
ipsec pki --pub --in clientKey.pem | ipsec pki --issue --cacert caCert.pem --cakey caKey.pem --dn "C=CN, O=strongSwan, CN=iphoneClient" --outform pem > clientCert.pem
```

很多博客里会说让生成p12证书，然而在iOS8以上好像并不需要这个东西。

将证书Copy到对应的目录下，使用apt-get来安装strongswan的话，证书需要copy到/etc目录下

``` bash
sudo mv caCert.pem /etc/ipsec.d/cacerts/
sudo mv serverCert.pem /etc/ipsec.d/certs/
sudo mv caKey.pem /etc/ipsec.d/private/
sudo mv serverKey.pem /etc/ipsec.d/private/
sudo mv clientKey.pem /etc/ipsec.d/private/
```

---

**配置strongswan**

因为我只需要让我的iphone能够翻墙，所以我这里只有iphone的配置。



编辑`/etc/ipsec.conf`

``` bash
# ipsec.conf - strongSwan IPsec configuration file

# basic configuration

config setup
	# strictcrlpolicy=yes
	uniqueids = never

# Add connections here.
conn %default
    left=%defaultroute
    ikelifetime=60m
    keylife=20m
    rekeymargin=3m
    keyingtries=1
    keyexchange=ikev2  # 协议是ikev2
    authby=secret  # 验证方式是密码，也可以用证书，证书配置起来比较麻烦，需要选用iphone支持的签名方法(懒得下证书了-，-#)

conn iOS_psk
    leftsubnet=0.0.0.0/0
    leftfirewall=yes
    right=%any
    rightsourceip=10.63.41.0/24  # 分配的内网ip
    auto=add
```



编辑`/etc/ipsec.secrets`

``` bash
# This file holds shared secrets or RSA private keys for authentication.

# RSA private key for this host, authenticating it to any other host
# which knows the public part.  Suitable public keys, for ipsec.conf, DNS,
# or configuration of other implementations, can be extracted conveniently
# with "ipsec showhostkey".

: PSK "<你的密码，尽量不要使用太奇怪的字符>"

```



编辑`/etc/strongswan.conf`，其他参数没有测试过，但是dns是一定需要的，不然的话上不了网，不知其他网友有没有这种情况。

``` bash
# strongswan.conf - strongSwan configuration file
#
# Refer to the strongswan.conf(5) manpage for details
#
# Configuration changes should be made in the included files

charon {
	load_modular = yes
        duplicheck.enable = no
        compress = yes
        dns1 = 8.8.8.8
        dns2 = 8.8.4.4
	plugins {
		include strongswan.d/charon/*.conf
	}
}

include strongswan.d/*.conf
```

---

**打开ip转发**

``` bash
sudo sed -s 's/^#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/g' /etc/sysctl.conf -i
sudo sysctl -p
```

---

**配置iptable**

注意里面配置的ip，和上面ipsec.conf配置的ip一致，这个就不用多解释了。

这里我为了解决iptable重启之后会失效的问题，用了iptables-save这个工具，找不到这个工具的也可以找其他工具帮忙。

``` bash
#!/bin/bash
iptables -A INPUT -p udp --dport 500 -j ACCEPT
iptables -A INPUT -p udp --dport 4500 -j ACCEPT
iptables -t nat -A POSTROUTING -s 10.63.41.0/24 -o eth0 -j MASQUERADE
iptables -A FORWARD -s 10.63.41.0/24 -j ACCEPT

# 保存iptables配置
iptables-save > /etc/iptables/rules.v4
```

---

自此，服务器端的配置都已经完成，启动strongswan我们就可以使用iphone翻墙了

``` bash
# 普通启动
ipsec start

# 输出日志的启动
ipsec start --nofork
```

---

**iPhone配置**

<img width="402" alt="4e11705d-1eec-4edc-929e-66861f25b70f" src="https://user-images.githubusercontent.com/3870517/32063455-d1b00fb6-ba3c-11e7-92ec-8bf7d6340a4b.png">

描述可以随便写，服务器和远程ID都填服务器的ip地址，或者域名。

密码就是上文所配置的那个密码。

然后点击完成，vpn的配置就完成了。

**配置文件**

但是直接配置账号密码，并不能让iphone在切换网络之后自动连接。要自动连接的话，需要使用配置文件。

配置文件的样例如下：

``` xml
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<!-- Read more: https://wiki.strongswan.org/projects/strongswan/wiki/AppleIKEv2Profile -->
<plist version="1.0">
    <dict>
        <!-- Set the name to whatever you like, it is used in the profile list on the device -->
        <key>PayloadDisplayName</key>
        <string>${PROFILE_NAME}</string>
        <!-- This is a reverse-DNS style unique identifier used to detect duplicate profiles -->
        <key>PayloadIdentifier</key>
        <string>${PROFILE_IDENTIFIER}</string>
        <!-- A globally unique identifier, use uuidgen on Linux/Mac OS X to generate it -->
        <key>PayloadUUID</key>
        <string>${PROFILE_UUID}</string>
        <key>PayloadType</key>
        <string>Configuration</string>
        <key>PayloadVersion</key>
        <integer>1</integer>
        <key>PayloadContent</key>
        <array>
            <!-- It is possible to add multiple VPN payloads with different identifiers/UUIDs and names -->
            <dict>
                <!-- This is an extension of the identifier given above -->
                <key>PayloadIdentifier</key>
                <string>${CONN_IDENTIFIER}</string>
                <!-- A globally unique identifier for this payload -->
                <key>PayloadUUID</key>
                <string>${CONN_UUID}</string>
                <key>PayloadType</key>
                <string>com.apple.vpn.managed</string>
                <key>PayloadVersion</key>
                <integer>1</integer>
                <!-- This is the name of the VPN connection as seen in the VPN application later -->
                <key>UserDefinedName</key>
                <string>${CONN_NAME}</string>
                <key>VPNType</key>
                <string>IKEv2</string>
                <key>IKEv2</key>
                <dict>
                    <!-- Hostname or IP address of the VPN server -->
                    <key>RemoteAddress</key>
                    <string>${CONN_HOST}</string>
                    <!-- Remote identity, can be a FQDN, a userFQDN, an IP or (theoretically) a certificate's subject DN. Can't be empty.
                     IMPORTANT: DNs are currently not handled correctly, they are always sent as identities of type FQDN -->
                    <key>RemoteIdentifier</key>
                    <string>${CONN_REMOTE_IDENTIFIER}</string>
                    <!-- Local IKE identity, same restrictions as above. If it is empty the client's IP address will be used -->
                    <key>LocalIdentifier</key>
                    <string></string>
                    <!--
                    OnDemand references:
                    https://developer.apple.com/library/mac/featuredarticles/iPhoneConfigurationProfileRef/Introduction/Introduction.html
                    -->
                    <key>OnDemandEnabled</key>
                    <integer>1</integer>
                    <key>OnDemandRules</key>
                    <array>
                        <dict>
                            <key>Action</key>
                            <string>Connect</string>
                        </dict>
                    </array>
                    <!-- The server is authenticated using a certificate -->
                    <key>AuthenticationMethod</key>
                    <string>SharedSecret</string>
                    <key>SharedSecret</key>
                    <string>${CONN_SHARED_SECRET}</string>
                    <!-- The client uses EAP to authenticate -->
                    <key>ExtendedAuthEnabled</key>
                    <integer>0</integer>
                </dict>
            </dict>
        </array>
    </dict>
</plist>
```

 里面有不少东西需要自己替换成自己的配置。所以最后我干脆写了一个脚本，用来配置这个东西。

配置一共有3个文件

``` bash
config.sh  # 用于配置环境，并生成iphone使用的.mobileconfig文件
generate-mobileconfig.sh  # iphone使用的.mobileconfig文件的模板，config.sh会调用这个脚本
iptables.sh  # 配置iptables的脚本
```

*config.sh*

``` bash
#!/bin/bash

HOST="<这里写服务器的ip，或者域名>"
export HOST=$HOST

ipsec pki --gen --outform pem > caKey.pem
ipsec pki --self --in caKey.pem --dn "C=CN, O=myNetwork, CN=myCA" --ca --outform pem > caCert.pem

ipsec pki --gen --outform pem > serverKey.pem
ipsec pki --pub --in serverKey.pem | ipsec pki --issue --cacert caCert.pem --cakey caKey.pem --dn "C=CN, O=myNetwork, CN=$host" --san="$host" --flag serverAuth --flag ikeIntermediate --outform pem > serverCert.pem

ipsec pki --gen --outform pem > clientKey.pem
ipsec pki --pub --in clientKey.pem | ipsec pki --issue --cacert caCert.pem --cakey caKey.pem --dn "C=CN, O=strongSwan, CN=iphoneClient" --outform pem > clientCert.pem

openssl pkcs12 -export -inkey clientKey.pem -in clientCert.pem -name "iphoneClient" -certfile caCert.pem -caname "myCA" -out clientCert.p12

sudo mv caCert.pem /etc/ipsec.d/cacerts/
sudo mv serverCert.pem /etc/ipsec.d/certs/
sudo mv caKey.pem /etc/ipsec.d/private/
sudo mv serverKey.pem /etc/ipsec.d/private/
sudo mv clientKey.pem /etc/ipsec.d/private/

sudo cp ipsec.conf /etc/
sudo cp ipsec.secrets /etc/
sudo cp strongswan.conf /etc/

sudo sed -s 's/^#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/g' /etc/sysctl.conf -i
sudo sysctl -p

./generate-mobileconfig.sh > $HOST.mobileconfig

```

*iptables.sh*

``` bash
#!/bin/bash
iptables -A INPUT -p udp --dport 500 -j ACCEPT
iptables -A INPUT -p udp --dport 4500 -j ACCEPT
iptables -t nat -A POSTROUTING -s 10.63.41.0/24 -o eth0 -j MASQUERADE
iptables -A FORWARD -s 10.63.41.0/24 -j ACCEPT
iptables-save > /etc/iptables/rules.v4
```

*generate-mobileconfig.sh*

``` bash
#!/bin/bash

# TODO: add regenerate shared secret option

# In normal cases, you will only need to pass the HOST of your server.
#[ "no${HOST}" = "no" ] && echo "\$HOST environment variable required." && exit 1
: ${PROFILE_NAME="${HOST} Profile"}
: ${PROFILE_IDENTIFIER=$(echo -n "${HOST}." | tac -s. | sed 's/\.$//g')}
: ${PROFILE_UUID=$(hostname)}
# These variable, especially CONN_UUID, are bind to per username,
# which currently, all users share the same secrets and configurations.
: ${CONN_NAME="${HOST}"}
: ${CONN_IDENTIFIER="${PROFILE_IDENTIFIER}.shared-configuration"}
: ${CONN_UUID=$(uuidgen)}
: ${CONN_HOST=${HOST}}
: ${CONN_REMOTE_IDENTIFIER=${HOST}}
CONN_SHARED_SECRET=$(grep "PSK" /etc/ipsec.secrets | sed 's/.*"\(.*\)"/\1/g')

cat <<EOF
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<!-- Read more: https://wiki.strongswan.org/projects/strongswan/wiki/AppleIKEv2Profile -->
<plist version="1.0">
    <dict>
        <!-- Set the name to whatever you like, it is used in the profile list on the device -->
        <key>PayloadDisplayName</key>
        <string>${PROFILE_NAME}</string>
        <!-- This is a reverse-DNS style unique identifier used to detect duplicate profiles -->
        <key>PayloadIdentifier</key>
        <string>${PROFILE_IDENTIFIER}</string>
        <!-- A globally unique identifier, use uuidgen on Linux/Mac OS X to generate it -->
        <key>PayloadUUID</key>
        <string>${PROFILE_UUID}</string>
        <key>PayloadType</key>
        <string>Configuration</string>
        <key>PayloadVersion</key>
        <integer>1</integer>
        <key>PayloadContent</key>
        <array>
            <!-- It is possible to add multiple VPN payloads with different identifiers/UUIDs and names -->
            <dict>
                <!-- This is an extension of the identifier given above -->
                <key>PayloadIdentifier</key>
                <string>${CONN_IDENTIFIER}</string>
                <!-- A globally unique identifier for this payload -->
                <key>PayloadUUID</key>
                <string>${CONN_UUID}</string>
                <key>PayloadType</key>
                <string>com.apple.vpn.managed</string>
                <key>PayloadVersion</key>
                <integer>1</integer>
                <!-- This is the name of the VPN connection as seen in the VPN application later -->
                <key>UserDefinedName</key>
                <string>${CONN_NAME}</string>
                <key>VPNType</key>
                <string>IKEv2</string>
                <key>IKEv2</key>
                <dict>
                    <!-- Hostname or IP address of the VPN server -->
                    <key>RemoteAddress</key>
                    <string>${CONN_HOST}</string>
                    <!-- Remote identity, can be a FQDN, a userFQDN, an IP or (theoretically) a certificate's subject DN. Can't be empty.
                     IMPORTANT: DNs are currently not handled correctly, they are always sent as identities of type FQDN -->
                    <key>RemoteIdentifier</key>
                    <string>${CONN_REMOTE_IDENTIFIER}</string>
                    <!-- Local IKE identity, same restrictions as above. If it is empty the client's IP address will be used -->
                    <key>LocalIdentifier</key>
                    <string></string>
                    <!--
                    OnDemand references:
                    https://developer.apple.com/library/mac/featuredarticles/iPhoneConfigurationProfileRef/Introduction/Introduction.html
                    -->
                    <key>OnDemandEnabled</key>
                    <integer>1</integer>
                    <key>OnDemandRules</key>
                    <array>
                        <dict>
                            <key>Action</key>
                            <string>Connect</string>
                        </dict>
                    </array>
                    <!-- The server is authenticated using a certificate -->
                    <key>AuthenticationMethod</key>
                    <string>SharedSecret</string>
                    <key>SharedSecret</key>
                    <string>${CONN_SHARED_SECRET}</string>
                    <!-- The client uses EAP to authenticate -->
                    <key>ExtendedAuthEnabled</key>
                    <integer>0</integer>
                </dict>
            </dict>
        </array>
    </dict>
</plist>
EOF

```

最后整个配置也只有我自己试用过是可以的。如果发现用不了，请自己上网找怎么解决。

生成了`.mobileconfig`文件后，使用`python -m SimpleHTTPServer`命令运行一个web服务，然后在iphone上用Safari打开这个文件，安装配置。然后你就有了一个可以自动连接的ikev2配置。

---

后记：iphone上我只使用过openvpn和ikev2个人感觉ikev2十分稳定，到目前为止没有出现过被block的情况。出门打po面基都十分方便，用instagram装逼也是妥妥的。

PS：我是蓝...
