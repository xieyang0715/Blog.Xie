---
title: 对象存储(radosgw)
date: 2021-04-06 08:45:13
---



# 对象存储

Object Storage Service. 对象存储服务，一般维护一个对象存储需要一个团队，一般使用流行的市面上的OSS即可 ：七牛云、百度云、阿里云、dropbox，....。

OSS把用户提交的数据，放一个平面管理，每个对象有自已的ID, data, metadata。只能使用api(http/https)访问。



## 三个基本概念

- [x] Users: 用户
- [x] Buckets, Containers：归组，或一个目录
- [x] Objects对象

S3: user, buckets, object

swift: Account(租户），user(子用户), container, object

radosRGW: tenant(租户)，user(对应s3的user)/subuser(对应swift的user),  bucket, object

ACL: read, write, readwrite, full-control

# [部署RGW](http://blog.mykernel.cn/2021/02/24/ceph%E7%B3%BB%E7%BB%9F%E9%83%A8%E7%BD%B2/)

## 创建Users

```bash
[cephadm@ceph-admin01 ceph-cluster]$  radosgw-admin user create --uid='s3user' --display-name='S3 Testing User'
{
    "user_id": "s3user",
    "display_name": "S3 Testing User",
    "email": "",
    "suspended": 0,
    "max_buckets": 1000,
    "subusers": [],
    "keys": [
        {
            "user": "s3user",
            "access_key": "EJAYFXH3RLQJ0GOB3QTL",
            "secret_key": "wbx8x40bHWjuHF1CS2UsBkBaKKtGEf4OlIgiHRwv"
        }
    ],
    "swift_keys": [],
    "caps": [],
    "op_mask": "read, write, delete",
    "default_placement": "",
    "default_storage_class": "",
    "placement_tags": [],
    "bucket_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "user_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "temp_url_keys": [],
    "type": "rgw",
    "mfa_ids": []
}

```

## 准备s3cmd

```bash
root@ceph-client:~# apt install s3cmd
```

## 配置s3cmd

```diff
+root@ceph-client:~# s3cmd --configure

Enter new values or accept defaults in brackets with Enter.
Refer to user manual for detailed description of all options.

Access key and Secret key are your identifiers for Amazon S3. Leave them empty for using the env variables.
+Access Key: EJAYFXH3RLQJ0GOB3QTL
+Secret Key: wbx8x40bHWjuHF1CS2UsBkBaKKtGEf4OlIgiHRwv
+Default Region [US]: 

Use "s3.amazonaws.com" for S3 Endpoint and not modify it to the target Amazon S3.
+S3 Endpoint [s3.amazonaws.com]: store01.mykernel.io:7480

Use "%(bucket)s.s3.amazonaws.com" to the target Amazon S3. "%(bucket)s" and "%(location)s" vars can be used
if the target S3 system supports dns based buckets.
+DNS-style bucket+hostname:port template for accessing a bucket [%(bucket)s.s3.amazonaws.com]: %+(bucket)s.store01.mykernel.io:7480

Encryption password is used to protect your files from reading
by unauthorized persons while in transfer to S3
+Encryption password: 
+Path to GPG program [/usr/bin/gpg]: 

When using secure HTTPS protocol all communication with Amazon S3
servers is protected from 3rd party eavesdropping. This method is
slower than plain HTTP, and can only be proxied with Python 2.7 or newer
+Use HTTPS protocol [Yes]: No

On some networks all internet access must go through a HTTP proxy.
Try setting it here if you can't connect to S3 directly
+HTTP Proxy server name: 

New settings:
  Access Key: EJAYFXH3RLQJ0GOB3QTL
  Secret Key: wbx8x40bHWjuHF1CS2UsBkBaKKtGEf4OlIgiHRwv
  Default Region: US
  S3 Endpoint: store01.mykernel.io:7480
  DNS-style bucket+hostname:port template for accessing a bucket: %(bucket)s.store01.mykernel.io:7480
  Encryption password: 
  Path to GPG program: /usr/bin/gpg
  Use HTTPS protocol: False
  HTTP Proxy server name: 
  HTTP Proxy server port: 0

+Test access with supplied credentials? [Y/n] Y
Please wait, attempting to list all buckets...
Success. Your access key and secret key worked fine :-)

Now verifying that encryption works...
Not configured. Never mind.

+Save settings? [y/N] Y
+Configuration saved to '/root/.s3cfg'  # 配置文件
```

## s3cmd管理radosgw

### 创建bucket

```bash
root@ceph-client:~# s3cmd mb s3://images
Bucket 's3://images/' created
```

### 列出bucket

```bash
root@ceph-client:~# s3cmd ls
2021-04-06 09:29  s3://images
```

### 上传图片

```bash
root@ceph-client:~# find /usr/share -iname "*.png" | head
/usr/share/plymouth/ubuntu-logo.png
/usr/share/pixmaps/htop.png
/usr/share/pixmaps/language-selector.png
/usr/share/pixmaps/debian-logo.png
/usr/share/gitweb/static/git-logo.png
/usr/share/gitweb/static/git-favicon.png
/usr/share/language-selector/data/language-selector.png
/usr/share/icons/Humanity/status/24/audio-volume-low.png
/usr/share/icons/Humanity/status/24/audio-volume-high.png
/usr/share/icons/Humanity/status/24/audio-volume-muted.png



root@ceph-client:~# s3cmd put /usr/share/icons/Humanity/status/24/audio-volume-muted.png s3://images/
upload: '/usr/share/icons/Humanity/status/24/audio-volume-muted.png' -> 's3://images/audio-volume-muted.png'  [1 of 1]
 609 of 609   100% in    0s   187.67 kB/s
root@ceph-client:~# s3cmd put /usr/share/gitweb/static/git-favicon.png s3://images/webimgs/1.jpg
upload: '/usr/share/gitweb/static/git-favicon.png' -> 's3://images/webimgs/1.jpg'  [1 of 1]
 115 of 115   100% in    0s     4.51 kB/s  done

```

### 列出上传的对象

```bash
root@ceph-client:~# s3cmd ls s3://images
                       DIR   s3://images/webimgs/
2021-04-06 09:31       609   s3://images/audio-volume-muted.png
root@ceph-client:~# s3cmd ls s3://images/webimgs/
2021-04-06 09:31       115   s3://images/webimgs/1.jpg
```

### 下载对象

```bash
root@ceph-client:~# s3cmd get s3://images/audio-volume-muted.png
download: 's3://images/audio-volume-muted.png' -> './audio-volume-muted.png'  [1 of 1]
 609 of 609   100% in    0s    60.17 kB/s  done
root@ceph-client:~# ls
audio-volume-muted.png  

```



