---
title: 使用 acme.sh 自动签证书部署到腾讯云
date: 2023-03-07 20:38:00
tags:
  - SSL
  - 云服务
categories:
  - 运维
---

人太懒就是这样的

# 直接上内容

签发指令

```shell
CF_Email="email@foobar.com" CF_Key="foobarfoobar" acme.sh --issue -d foobar.com -d \*.foobar.com --dns dns_cf --keylength ec-256

acme.sh --installcert -d foobar.com --ecc --key-file /opt/foobar/key.pem --fullchain-file /opt/foobar/cert.pem --reloadcmd "安装证书一把梭.sh"
```

# 如何续签后执行部署脚本？

在 acme 安装证书后面加上 `--reloadcmd "安装证书一把梭.sh"`

其内容可以是任何指令

比如说我要执行一个 python 脚本

```shell
acme.sh --installcert -d foobar.com --ecc --key-file /opt/foobar/key.pem --fullchain-file /opt/foobar/cert.pem --reloadcmd "python a.py"
```

接下来写脚本内容（我要上传到腾讯云

```python
# a.py

import json
from tencentcloud.common import credential
from tencentcloud.common.exception.tencent_cloud_sdk_exception import TencentCloudSDKException
from tencentcloud.ssl.v20191205 import ssl_client, models as ssl_models
from tencentcloud.cdn.v20180606 import cdn_client, models as cdn_models
from tencentcloud.live.v20180801 import live_client, models as live_models

try:
    # 为了保护密钥安全，建议将密钥设置在环境变量中或者配置文件中，请参考本文凭证管理章节。
    # 硬编码密钥到代码中有可能随代码泄露而暴露，有安全隐患，并不推荐。
    cred = credential.Credential("id", "key")

    # 获取证书
    certPubKey = ''
    certpriKey = ''
    with open('/opt/foobar/cert.pem', 'r') as f:
        certPubKey = f.read()
    with open('/opt/foobar/key.pem', 'r') as f:
        certpriKey = f.read()

    # 上传证书
    client = ssl_client.SslClient(cred, '')
    req = ssl_models.UploadCertificateRequest()
    params = {
        'CertificatePublicKey': certPubKey,
        'CertificatePrivateKey': certpriKey,
        'CertificateType': 'SVR',
    }
    req.from_json_string(json.dumps(params))
    resp = client.UploadCertificate(req)
    print(resp.to_json_string())
    certID = resp.CertificateId

    # 部署CDN证书
    client = cdn_client.CdnClient(cred, '')
    req = cdn_models.UpdateDomainConfigRequest()
    params = {
        'Https': {
            'Switch': 'on',
            'CertInfo': {
                'CertId': certID,
            }
        },
    }
    req.from_json_string(json.dumps(params))
    req.Domain = 'foobar.com'
    resp = client.UpdateDomainConfig(req)
    print(resp.to_json_string())

    # 部署直播证书
    client = live_client.LiveClient(cred, '')
    req = live_models.ModifyLiveDomainCertBindingsRequest()
    params = {
        'DomainInfos': [
            {
                'DomainName': 'live.foobar.com',
                'Status': -1,
            }
        ],
        'CloudCertId': certID,
    }
    req.from_json_string(json.dumps(params))
    resp = client.ModifyLiveDomainCertBindings(req)
    print(resp.to_json_string())

except TencentCloudSDKException as err:
    print(err)
```

# 搞定开摆！

再也不需要自己上传部署了
