title: fix-docker-x509-error
date: 2016-08-03 02:24:59
tags: docker
---

1. 问题描述

由于公司内部证书被IT部门修改，我们在使用docker pull某些镜像的时候会抛出x509错误 x509: certificate signed by unknown authority。
例如，我在尝试用docker-compose获取最新版本的spark时会显示：

```
abuuu-VirtualBox# sudo docker-compose up
Pulling master (gettyimages/spark:latest)...
latest: Pulling from gettyimages/spark
357ea8c3d80b: Pulling fs layer
c6cf625461b9: Pulling fs layer
06fd4f43f066: Pulling fs layer
dd98390795f4: Waiting
36769b1579ad: Waiting
b2e57763c10f: Waiting
ERROR: error pulling image configuration: Get https://dseasb33srnrn.cloudfront.net/registry-v2/docker/registry/v2/blobs/sha256/4e/4ef9bff9a39ea255de6945d1480a771b4785b17a0da492fd6427e98ec5d624dd/data?Expires=1470184587&Signature=TpEb2htK0E8yUKVinb03onAc35rMqzC4JPJeWnXQ1DkmFifVORmP9-Vusc8vtZjFG3yCyWgfIL8zRVLhmj3koVtb~QLcx5eHcmHprzj6nxXt~GC-MuUT91t65Q2eOqwQDNQAwlcPxP9moxggWmoGQaHyII0bIwBtvdZ7GiUnE0w_&Key-Pair-Id=APKAJECH5M7VWIS5YZ6Q: x509: certificate signed by unknown authority
```

我们可以看出，在访问dseasb33srnrn.cloudfront.net时产生了证书验证错误。

2. 解决方法

参考http://www.cnblogs.com/sting2me/p/5596222.html上描述的方式，用openssl访问该网站，并将被替换后的证书保存至本地并用update-ca-certificates更新：

代码如下(在Ubuntu14.04下测试通过)

```
echo -n | openssl s_client -showcerts -connect dseasb33srnrn.cloudfront.net:443 2>/dev/null | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > /usr/local/share/ca-certificates/cloudfront.crt
update-ca-certificates
sudo service docker restart
```

然后重新执行就会发现成功了，从此以后就可以在上班的时候轻松摸鱼了（误）。
