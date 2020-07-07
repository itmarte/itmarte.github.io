---
title: RSA加解密钥对生成
tags:
  - linux
  - openssl
categories:
  - linux
  - openssl
date: 2020-07-01 19:55:55
---

\#creating rsa private key

```shell
openssl genrsa -out rsa_private_key.pem 2048
```



\#convert private key into PKCS#8 format

```shell
openssl pkcs8 -topk8 -inform PEM -in rsa_private_key.pem -outform PEM -nocrypt -out private_pkcs8.pem
```



\#get public key from private key (X509 format)

```shell
openssl rsa -in rsa_private_key.pem -pubout -out rsa_public_key.pem
```