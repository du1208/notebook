```diff
+ red

- green
```



**Trojan** 是近来比较热门的一款代理工具，其设计理念同其他工具有些差别，会将代理数据伪装成标准的 HTTPS 流量，以避免受到 GFW 的影响，因为 GFW 还不至于胆大到阻挡正常的 HTTPS 流量，引起互联网正常业务的故障。GFW 的确强大到可以折数据包根据协议特征码来识别 SS 和 V2Ray 的流量，但是如果代理的数据就是普通的 HTTPS 数据包的话，反倒可以绕过 GFW 的审核。

Trojan 用C 和 C++ 语言开发，执行效率高，Linux 平台和 Windows 平台均表现良好。目前最新的发行版版本为1.13.0，对于特定的 Linux 发行版，Trojan 还可以通过包管理工具安装预编译版本，使用起来就更为方便。

本教程将以 Debian 10 为例，详细讲解如何使用 IP 地址进行自签名，免除使用实际域名的过程，来安装和使用 Trojan，并将预安装版本升级至最新版，以及浏览器的设置来科学上网。

**1] 系统环境说明**

前面说过，Trojan 会把代理数据伪装成 HTTPS 流量，这就要求数据包的加密与标准的 HTTPS 数据一致，网上很多 Trojan 教程都会指导大家申请一个域名，这样做的目的是可以通过域名签名工具生成服务器域名证书，从而完成 Trojan 的配置。这样做的好处就是签名证书是真实可靠的，可以被浏览器接受，缺点就是签名时效都比较短，需要定期更新证书。

事实上这个签名证书也可以自行生成，并且可以直接使用主机的 IP 地址进行签名。这个过程相对复杂，但好处也非常明显，就是可以自行设置签名证书有效期，并且不需要再另行申请域名。

采用 Debian 10 的原因是其包管理工具中已经包含 Trojan 预编译版本，使用包管理工具即可安装，避免依赖包安装不全的情况。

**2] 生成 IP  地址自签名证书**

先大概说下签名证书的原理，想要生成签名证书，需要一个 CA 服务器，然后 Web 服务器会在此 CA 服务器上注册并生成自己的公钥和私钥，而浏览器访问此 Web 服务器时，会首先获得 Web 服务器的证书，证书中包含 CA 证书颁发机构信息、 Web 服务器（证书申请者）信息、公钥以及签名，客户端验证证书的有效性后，会使用公钥将请求信息加密，并生成同 Web 服务器的共享密钥，Web 服务器收到请求信息后，使用私钥解密信息，并通过共享密钥将响应信息加密返回给客户端，客户端收到信息后通过共享密钥解密数据，从而完成 SSL 连接的建立。

如果使用 IP 地址自签名，那么就需要自行搭建 CA 证书服务器，然后由此证书服务器生成用于 SSL 连接的服务器公钥和私钥，最后，客户端通过 CA 证书服务器的公钥来验证其生成的 Web 服务器公钥有效性。显然，这里的 CA 证书服务器、Web 服务器都由 VPS 主机来充当。

这部分内容可能有些复杂，但是并不影响实际的自签名过程，不理解也没有关系，按以下步骤操作即可。

更新系统并安装工具包
```
apt-get update
apt-get install gnutls-bin
```
新建两个文件”ca.txt”和”server.txt”作为签名模板，文件内容如下：

ca.txt
```
cn = "192.168.1.1"
organization = "GlobalSign RULTR"
serial = 1
expiration_days = 3650
ca
signing_key
cert_signing_key
crl_signing_key

```
server.txt
```
cn = "192.168.1.1"
organization = "GlobalSign RULTR"
expiration_days = 3650
signing_key
encryption_key
tls_www_server

```
其中”ca.txt”是用于生成 CA 证书服务器签名的模板文件，而”server.txt”是用于生成 Trojan 使用的服务器签名模板。

模板中”cn”是自己主机的公网地址，根据实际情况填写，”organization”是组织名称，可以根据喜好自行设置，”expiration_days”表示证书的有效期，单位为天，示例设置有效期设置为10年，应该够用；其他内容不需要修改。

现在，根据证书模板生成 CA 证书和 Trojan 服务器证书：
```
certtool --generate-privkey --outfile ca-key.pem
certtool --generate-self-signed --load-privkey ca-key.pem --template ca.txt --outfile ca-cert.pem
certtool --generate-privkey --outfile trojan-key.pem
certtool --generate-certificate --load-privkey trojan-key.pem --load-ca-certificate ca-cert.pem --load-ca-privkey ca-key.pem --template server.txt --outfile trojan-cert.pem
```
命令执行完成后，会生成四个证书文件，其中”ca-key.pem”是 CA 证书的私钥；”ca-cert.pem”是 CA 证书服务器的公钥；”trojan-key.pem”是申请到的私钥，”trojan-cert.pem”是申请到的公钥。证书文件生成成功后，则进行一下步。

**3] 安装和配置 Trojan**

安装 Trojan 就采用包管理工具安装
```
apt-get install trojan
```
安装成功后，会在”/etc/trojan”目录生成一个”config.json”配置文件，这个配置文件非常重要，trojan 会根据配置文件选择工作在服务器模式还是客户端模式，现在将 trojan 使用的证书文件拷贝到该目录：
```
cp trojan-cert.pem trojan-key.pem /etc/trojan
```
然后修改该配置文件内容如下：
```
{
    "run_type": "server",
    "local_addr": "0.0.0.0",
    "local_port": 443,
    "remote_addr": "127.0.0.1",
    "remote_port": 80,
    "password": [
        "Your_Password_Here"
    ],
    "log_level": 1,
    "ssl": {
        "cert": "/etc/trojan/trojan-cert.pem",
        "key": "/etc/trojan/trojan-key.pem",
        "key_password": "",
        "cipher": "ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA
-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256",
        "prefer_server_cipher": true,
        "alpn": [
            "http/1.1"
        ],
        "reuse_session": true,
        "session_ticket": false,
        "session_timeout": 600,
        "plain_http_response": "",
        "curves": "",
        "dhparam": ""
    },
    "tcp": {
        "prefer_ipv4": false,
        "no_delay": true,
        "keep_alive": true,
        "fast_open": false,
        "fast_open_qlen": 20
    },
    "mysql": {
        "enabled": false,
        "server_addr": "127.0.0.1",
        "server_port": 3306,
        "database": "trojan",
        "username": "trojan",
        "password": ""
    }
}
```
仅需要修改标红部分的内容即可，将”Your_Password_Here”部分修改为自己喜欢的密码，这是 trojan 客户端同服务器论证的密码。如果设置不出，也可以使用如下命令通过系统随机生成一个字符串当作密码：
```
cat /proc/sys/kernel/random/uuid
```
如果此时启动 trojan 服务，系统会提示访问被拒绝，查看其服务配置文件”/lib/systemd/system/trojan.service”，发现其中内容中设置了启动 trojan 的用户：
```
[Unit]
Description=trojan
Documentation=man:trojan(1) https://trojan-gfw.github.io/trojan/config https://trojan-gfw.github.io/trojan/
After=network.target mysql.service mariadb.service mysqld.service

[Service]
Type=simple
StandardError=journal
User=trojan
AmbientCapabilities=CAP_NET_BIND_SERVICE
ExecStart=/usr/bin/trojan /etc/trojan/config.json
ExecReload=/bin/kill -HUP $MAINPID

[Install]
WantedBy=multi-user.target
```
这是因为作者出于安全性的考虑，将 trojan 用户设置为较低权限的用户，但是如果系统没有该用户，或者用户对于配置文件目录没有访问权限，则会报权限不足错，trojan 启动失败。此处，我已经将原来的”User=nobody”修改为”User=trojan”，方便管理。如果不考虑安全因素的话，可以将此行删除，以 root 用户启动 trojan 即可。

既然配置文件中使用了非 root 用户，我们使用如下命令创建该用户并对配置目录授权：
```
groupadd -g 54321 trojan
useradd -g trojan -s /usr/sbin/nologin trojan
chown -R trojan:trojan /etc/trojan
```
现在，启动 trojan 并设置为开机启动：
```
systemctl start trojan
systemctl enable trojan
systemctl status trojan
```
如果状态信息显示 trojan 已经正常启动了，则可以进行下一步，如启动不成功，则需要根据错误提示查找原因。

**4] trojan 客户端配置(windows)**

trojan 是可执行程序，也具有 Windows 平台的发行版，可以从 GitHub 网站下载，下载完成后，需要将主机生成 的 CA 公钥文件拷贝到 trojan 文件所在目录，然后修改配置文件”config.json”为如下内容：
```
{
    "run_type": "client",
    "local_addr": "127.0.0.1",
    "local_port": 1080,
    "remote_addr": "Trojan_Server_IP_Here",
    "remote_port": 443,
    "password": [
        "Your_Password_Here"
    ],
    "log_level": 1,
    "ssl": {
        "verify": true,
        "verify_hostname": true,
        "cert": "ca-cert.pem",
        "cipher": "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-RSA-AES128-SHA:ECDHE-RSA-AES256-SHA:RSA-AES128-GCM-SHA256:RSA-AES256-GCM-SHA384:RSA-AES128-SHA:RSA-AES256-SHA:RSA-3DES-EDE-SHA",
        "sni": "",
        "alpn": [
            "h2",
            "http/1.1"
        ],
        "reuse_session": true,
        "session_ticket": false,
        "curves": ""
    },
    "tcp": {
        "no_delay": true,
        "keep_alive": true,
        "fast_open": false,
        "fast_open_qlen": 20
    }
}
```
需要修改的内容也不多，”remote_addr”指的是远端 trojan 服务器地址或域名，此处输入其  IP 地址；”password”设置同服务端一致；”cert”为 CA 服务器公钥，此处用从 trojan 服务器拷贝来的 CA 公钥。设置完成后，就可以双击”trojan.exe”启动程序了。
