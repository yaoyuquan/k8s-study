# docker pull配置代理
1. 创建目录
```
mkdir -p /etc/systemd/system/docker.service.d
```
2. 创建文件http-proxy.conf
```
[Service]
Environment="HTTP_PROXY=http://172.16.217.1:1080/"
```
3. 创建文件https-proxy.conf
```
[Service]
Environment="HTTPS_PROXY=https://172.16.217.1:1080"
```
4. 刷新配置
```
systemctl daemon-reload
systemctl restart docker
```

