**手动安装Trojan Linux客户端**

下载Trojan客户端官方版本(GitHub)：

```
cd /usr/src && wget https://github.com/trojan-gfw/trojan/releases/download/v1.16.0/trojan-1.16.0-linux-amd64.tar.xz
```
解压Trojan文件
```
tar xvf trojan-1.15.1-linux-amd64.tar.xz
```
打开配置文件
```
cd /usr/src/trojan
vi config.json
```
按i进入编辑模式
```
run_type 修改为 “client”
local_port 修改为 1080
remote_addr 修改为 vpn.xxx.cn
remote_port 修改为 443
password 修改为 [“123456”] trojan服务端验证密码
```

新建文件位置 /etc/systemd/system/trojan.service,配置trojan service，内容如下：

```
[Unit]
Description=trojan
After=network.target

[Service]
Type=simple
PIDFile=/usr/src/trojan/trojan.pid
ExecStart=/usr/src/trojan/trojan -c /usr/src/trojan/config.json -l /usr/src/trojan/trojan.log
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=1s

[Install]
WantedBy=multi-user.target
```
