**手动安装Trojan Linux客户端**

下载Trojan客户端官方版本(GitHub)：

```
cd /usr/src && wget https://github.com/trojan-gfw/trojan/releases/download/v1.16.0/trojan-1.16.0-linux-amd64.tar.xz
```
解压Trojan文件
```
tar xvf trojan-1.16.0-linux-amd64.tar.xz
```
编辑配置文件,修改相应的端口和密码
```
nano /usr/src/trojan/config.json
```

新建配置文件
```
nano /etc/systemd/system/trojan.service
```

粘贴如下内容

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

原文连接 https://blog.csdn.net/heroguo007/article/details/129858062
