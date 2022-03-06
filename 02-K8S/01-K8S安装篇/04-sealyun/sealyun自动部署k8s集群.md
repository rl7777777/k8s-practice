## 安装（`待补充`）

```
sealos init --passwd '********' \
	--master 10.0.0.11  --master 10.0.0.12  --master 10.0.0.13  \
	--pkg-url /root/kube1.22.0.tar.gz \
	--version v1.22.0 \
	--podcidr 172.16.0.0/12 \
	--svccidr 192.168.0.0/16 \
	--cert-sans apiserver.l2c.cn
```

| 配置信息    | 说明              |
| ----------- | ----------------- |
| 系统版本    | CentOS7.9         |
| 内核        | kernel-ml-4.19.12 |
| K8S版本     | 1.23.*            |
| Docker版本  | 20.10.*           |
| 主机网段    | 10.0.0.0/24       |
| Pod网段     | 172.16.0.0/12     |
| Service网段 | 192.168.0.0/16    |
