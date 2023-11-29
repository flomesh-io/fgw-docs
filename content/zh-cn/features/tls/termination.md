---
title: "TLS 卸载"
description: "TLS卸载是在负载均衡器或网关上终止TLS连接，解密流量后将其传送到后端服务器，从而减轻后端服务器的加密和解密负担。"
weight: 1
---

## 介绍

TLS 卸载是一种网络架构模式，它允许网络设备或服务器在接收到加密的传入网络连接后，解密并处理传输的数据，然后再将数据转发到后端服务器。这种模式可以在负载均衡、安全性和性能方面提供优势。与之相对的是 [TLS 透传](/features/tls/passthrough/)。

在 TLS 卸载的架构中，通常有一台前端设备（如负载均衡器、反向代理服务器）负责处理客户端的加密连接、解密数据，并根据特定规则将请求分发到后端服务器。这样可以减轻后端服务器的计算负担，因为它们不需要处理加密和解密操作。这里，FGW 可以承担该前端设备的角色。

## 前置条件

- Pipy（版本 >= {{< param min_pipy_version >}}）
- FGW Repo（版本 >= {{< param fgw_version >}}）
- 后端服务

## 配置说明

配置以下参数以启用和配置 TLS 卸载：

  * `TLSModeType`：定义 TLS 工作模式，设置为 "Terminate" 启动 TLS 卸载）。
  * `mTLS`：定义是否启用双向 TLS，增加额外的安全层来验证客户端的身份。
  * `Certificates`：包括以下三个配置项来配置 TLS 证书信息：
    * `CertChain`：服务器的证书链，通常开始于服务器的证书，后跟任何中间证书。
    * `PrivateKey`：用于与证书链中的服务器证书配对的私钥。
    * `IssuingCA`：用于签名证书链中的服务器证书的证书机构（CA）证书。

### 示例

```json
{
  "TLS": {
    "TLSModeType": "Terminate",
    "Certificates": {
      "CertChain": "—–BEGIN CERTIFICATE—–\n...",
      "PrivateKey": "—–BEGIN RSA PRIVATE KEY—–\n...",
      "IssuingCA": "—–BEGIN CERTIFICATE—–\n..."
    }
  }
}
```

## 配置

1. 要让负载均衡器支持 HTTPS 配置，我们需要先签发 CA 证书和服务器证书。

```shell
openssl genrsa 2048 > ca-key.pem

openssl req -new -x509 -nodes -days 365000 \
   -key ca-key.pem \
   -out ca-cert.pem \
   -subj '/CN=flomesh.io'

openssl genrsa -out server-key.pem 2048
openssl req -new -key server-key.pem -out server.csr -subj '/CN=example.com'
openssl x509 -req -in server.csr -CA ca-cert.pem -CAkey ca-key.pem -CAcreateserial -out server-cert.pem -days 365
```

执行上面的命令，我们会得到如下几个文件。

```
ca-cert.pem
ca-cert.srl
ca-key.pem
server-cert.pem
server-key.pem
server.csr
```

> 可以使用命令 `awk 'NF {sub(/\r/, ""); printf "%s\\n",$0;}' ca-cert.pem` 来输出单行。

2. 设置监听端口的协议，参考 [监听端口配置文档](/reference/configuration/#2-监听端口配置listeners)，我们选择 `TLS` 作为端口的协议。

```json
{
  "Listeners": [
    {
      "Protocol": "TLS",
      "Port": 8443,
      "TLS": {}
    }
  ]
}
```

3. TLS 证书的配置，可以参考文档 [TLS 配置](/reference/configuration/#22-tls)，将上面生成的 `server-cert.pem`、`server-key.pem`、`ca-cert.pem` 分别配置在 `CertChain`、`PrivateKey`、`IssuingCA` 字段。功能模式 `TLSModeType` 支持 `Terminate` 和 `Passthrough` 两种类型。`Passthrough` 在 [HTTPS 负载均衡](/features/tls/passthrough/) 中有使用，这里我们使用 `Terminate` 类型。

```json
{
  "TLS": {
    "TLSModeType": "Terminate",
    "mTLS": false,
    "Certificates": [
      {
        "CertChain": "-----BEGIN CERTIFICATE-----\nMIICpTCCAY0CCQD9vpm9qHfkjTANBgkqhkiG9w0BAQsFADAVMRMwEQYDVQQDDApm\nbG9tZXNoLmlvMB4XDTIzMDgwOTA5MDUwNVoXDTI0MDgwODA5MDUwNVowFDESMBAG\nA1UEAwwJbG9jYWxob3N0MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA\nv76yA1Zm5wKgfZjVKqNdEKd6Ejf8aQNHyk1gY8IKvWt2AeyqVAOqOjas/Pj8uVaG\nOrxfIAqqT1U5QF/WEPgUdxBgKTt2rXFr8f27bTmNQ6Z9irKaJx9BSyK7u6bvv9x6\nMgKxjtq43712R0W8gelfI3412KpYtpKTecMX2f3scCXNnksQUx7F35gg8kGcCLGN\nWsrT7+Jmdp6QP+S2fTLKr3qDEmZZMvRx/tKIR6WoZEhs7pnj6T3SLkK4pBjhwnxK\nclgJDBCG9UM0KNqmAnGMkSFKPTGs1x02eYhCf9+Hk3oXMi6DcWt7GH+Efmat/k4d\nNidOjJRBJuhMOZD+IhnxCwIDAQABMA0GCSqGSIb3DQEBCwUAA4IBAQAiWpzIROIN\nVySnxOm8Ua4E29c3GxB6KDOIpwxBizOW66wGA7zCfa8j/QgfTLV7q+hBj964PZJJ\nkK/Wuh8H99rzpQToOxWuFfBgNmPezFN/dSE5k+PQ6mItesWTtfDqFJIod4yIWoDf\nq9dbnOs0c3WGexP4YMo5WW9v/TKWfVgcc83aPsVnO5YpJC8y7VUDiyGKJW6R/k4y\n16fsrX4V1Gpj6ivFenG9OkftViff1/JuattMr3wq994SGlAggWL+H93CkyofbW+U\n3BVI4750Y0MYuDpt4DyI2zsKe2N5Ga7lNTfbj6tTdrfHkz0jdABDXc5lQr6ahTNO\nBvvV2Dpr0ozn\n-----END CERTIFICATE-----\n",
        "PrivateKey": "-----BEGIN RSA PRIVATE KEY-----\nMIIEowIBAAKCAQEAv76yA1Zm5wKgfZjVKqNdEKd6Ejf8aQNHyk1gY8IKvWt2Aeyq\nVAOqOjas/Pj8uVaGOrxfIAqqT1U5QF/WEPgUdxBgKTt2rXFr8f27bTmNQ6Z9irKa\nJx9BSyK7u6bvv9x6MgKxjtq43712R0W8gelfI3412KpYtpKTecMX2f3scCXNnksQ\nUx7F35gg8kGcCLGNWsrT7+Jmdp6QP+S2fTLKr3qDEmZZMvRx/tKIR6WoZEhs7pnj\n6T3SLkK4pBjhwnxKclgJDBCG9UM0KNqmAnGMkSFKPTGs1x02eYhCf9+Hk3oXMi6D\ncWt7GH+Efmat/k4dNidOjJRBJuhMOZD+IhnxCwIDAQABAoIBACjXLkVltt9HgPWf\ngu/lAeKVOXv97sZTS4w8dOZqoyz7YZRBW3ovmadyk+ACDJpRYp/KFZzWiLYDGgGr\nKAZPQNSnaUP/BWUl/m75s10tX/hj0uOi7RCeKKMfT8tFYFWGWYSjbDxYO/5z9Why\n4xbspTYDIOb4SZMBn2XU9xSYcC7mj/MaFr5bkMjLg+SMnG/QUgWoMA3F/hbmzrm2\n+5cqDY6Ew6EQk8IGRKRI3BrrgWrtljTcfDahLfA9Lu1+dE0ABkXUFuJ8ilgKunHi\nmo+61TPbVpo5A1hX3iexA1b38E8e2vAGzL8Lf4lAAxaJ823JmVqTJ28OZR+MFFXD\nXG05gyECgYEA5qS4aYHZM4/pvc1VIoLafWdH/nlbYQK9D6bBh3ICm2MGWWdta/nd\nB1bI7JT2J3lpb/mPK/pr8CXn60whO24XY18gU8A5HOlPO1l3QAHzIRLQFz11nb1M\nnwNen+u4zYEwnQPJSoWhh4S3686fY0OEUsxhc92x0QvkapcA429kTLECgYEA1NM0\nwbHjKNpbwW0P0KVHP2Cbwy3nB9topwJ8A6sT7cedjViDuP/ontvM7yosjc1YMN4t\noqg0tCPFGCfyoOSSnikEPBSYZzGCxaZ0K7tBWdHl+oDVNdPBGqIZzr2JypNpcRr6\nkH6gzjTFey/OJsVVOQPZ9D6TuMkk7mJAi11gmHsCgYEAyCUy3mPOvv7ooEtZ0Ivq\n3B3PDNX05RdCRx23HTljZ8Ij1Vt6SdPW6TJ3Q0302cZzJ7dRdaFnH0tVmQtEX1Um\nuJXo8KSDK0KO/fqiEAphGFdB+pjbwtltbyO2bmJYyQSN0gNiHugdhwM1s0xnZfVG\nE6/F9YzxbG28dn65R6P3TtECgYArjDQFVkLm/xc7Uvejd85GV5xHqcLWRrz5P3bk\nwULIqsnAPFZnqmWM6+jZH0YSlevvw+aOm+B847zWnoX1ChA+MKJfMM+mfekGTHME\n58INgPeP9ICsDPI8YuLo/LuPKe6vaBfRLTf2ObIW7MdAA6zWh8U3Rv6vFulppc0T\nNz4mtQKBgDApvH3mB5mCwh/Go0rKXzpOPfMKPip0/c65bkqzFqTjgFRG/A8gNMgD\ns9eGq8Gu+EO4ZvJvJHM/wf1d6vZ4kD0A1Cg/7VC7auy49dyI+A2J+1p/ebBjr3Dt\nPAgRO1uDYM6t7gstO4RddL3+fNW6whJ/QQRBwKJkwl8vVutROn7m\n-----END RSA PRIVATE KEY-----\n",
        "IssuingCA": "-----BEGIN CERTIFICATE-----\nMIICqDCCAZACCQCnQtjr6YkElTANBgkqhkiG9w0BAQsFADAVMRMwEQYDVQQDDApm\nbG9tZXNoLmlvMCAXDTIzMDgwOTA5MDUwNVoYDzMwMjIxMjEwMDkwNTA1WjAVMRMw\nEQYDVQQDDApmbG9tZXNoLmlvMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKC\nAQEAyjCOniAGJ/Afd99lrzVHVHnba3JM3mTuWAhfgXe/0A+RQejTpi/6XjBzWWsv\nmSX+ujtsEZWNZh8UeNtMwtS2ozM1gEte/jnIDBDl8tI7ZU78C3DGi9YvyXxUi6TO\nOJXtU3Z9Q2pKlusnLxUfqaXtq6cEGnU8x/62MOvoBxQ68B6ekuQZdA4i54K2S7KI\ngt+ROm2CD6iIr5vK6HouyS7LM4TECli1O5aFPhR3+PVVduiauhrcIhgELRpC84+e\n5VwtC4p7hZwQbtiOaXfaKCPFbzhdL2lzHJEDao58VpNMyE+nUI8uAtWjWvHXkD2t\nqSAqb8BPFAVAdWMmOeloWTZn8wIDAQABMA0GCSqGSIb3DQEBCwUAA4IBAQBqKsi2\nWnVFTRJ3+tJU0FX+rTiogYYxzFfhiJtwS6D/RRXRYOWIXRkhJWRAu+Gm4D2+yF9C\nSngD3dU9koG6N59qwnQ3nnRLfxvGltjk4VjDIiyJU1RU/3T4jDVXIwRB090j1/Se\nrra1WTKwqwm3EWoVbUNCdRsetRirSylqqYApfvWqeRewP7znX9MhfU0uncLOuTUe\nHpBOynUA48e5pxqBOeLS1iICuvixyqJHxLC/aIurpJm05CVNnjQGW2IPFBKJ9kb9\n+w1EC3kfDg9UQEK9QHh56bMUfBl2njWuBIw2AZ/lZe2yHSkauv2FCXfi6T3pO/OW\n+TdumkRp5puMKcn7\n-----END CERTIFICATE-----\n"
      }
    ]
  }
}
```

4. 接下来是设置端口 `8443` 的路由规则，参考文档 [TCP 协议端口号 Terminate 路由规则配置](/reference/configuration/#34-端口号配置protocol-tls-tlsmodetype-terminate-的配置格式)。其中键的内容为 TLS 证书的 SNI 名字，值为后端服务名。此处，我们使用自签发证书时使用的 `example.com`；值为服务 `backendService1`。

```json
{
  "RouteRules": {
    "8443": {
      "example.com": {"backendService1":100}
    }
  }
}
```

5. 配置服务端点，参考文档 [服务配置](/reference/configuration/#4-服务配置services)。

```json
{
  "Services": {
    "backendService1": {
      "Endpoints": {
        "127.0.0.1:8081": {
          "Weight": 100
        },
        "127.0.0.1:8082": {
          "Weight": 100
        }
      }
    }
  }
}
```

6. 不要忘记插件链的配置，本文档主要介绍 TLS 卸载，这里只需要引入 `TLSTerminate` 插件链。完整的插件配置可以参考文档 [完整的插件配置](/reference/plugin/#完整配置)。

```json
{
  "Chains": {
    "TLSTerminate": [
      "common/tls-termination.js",
      "common/consumer.js",
      "tls/forward.js"
    ]
  }
}
```

7. 最后完整的配置如下，使用其替换 FGW 工程中的 `pjs/config.json`。

```json
{
  "Listeners": [
    {
      "Protocol": "TLS",
      "Port": 8443,
      "TLS": {
        "TLSModeType": "Terminate",
        "mTLS": false,
        "Certificates": [
          {
            "CertChain": "-----BEGIN CERTIFICATE-----\nMIICpzCCAY8CCQD9vpm9qHfkjjANBgkqhkiG9w0BAQsFADAVMRMwEQYDVQQDDApm\nbG9tZXNoLmlvMB4XDTIzMDgwOTEzMzAwMloXDTI0MDgwODEzMzAwMlowFjEUMBIG\nA1UEAwwLZXhhbXBsZS5jb20wggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIB\nAQC8PVeWrdUW/daf7AWv1nGaxd+tVLe95RDk+A6rOmr5uoFBdO2OKfdYg3cNr3ti\nE2XbcBa00lrdhxMHANvNtvB+YApmyjXzLQW0XoJe6H6B28Q2RiSvlt4ek1w4h5j8\nUNGmKDZvL9PgPcVecKMqikHjsoPxKO/IdcB0SZ1sEnY6mQrFStVJI6ReUFAVIxK6\nFnbgJhhslU22+vF6hWvjdljl6YDyIpeuh+hjyGY6opFUv6hs7oHKvxnSPXb+c/Qm\nKfWfQiOswKJhgZcEeFvOfYrMNtCPXgF3TDHaAHbJ0+WPCllDlvNtCyFIcN8yD5JB\nyuPmXVJYP+Wg3ac/9mxLk+D/AgMBAAEwDQYJKoZIhvcNAQELBQADggEBABQpLMo+\nD4E/amfDxJN2oDo0Q8SA4H4uTqDgJdIL9iy91CmjMbiK1vrw7TSNoxjn33ds6bBt\n0xqsc//ckrgFSrUzqbkr7FYhLEd9Mwl4IaXwl6tk4IRzVEUMt+cRC4qadXd8uYVZ\ntbVMoMQ0vi3OEJhemb8eGB6yVhufCw7535oU0Us1lDegQ3nTp+jf7LYMLgbQKk1v\nqrvoSx5kgwXKHIDJ7jYHMtm2KH2H18274XM+WH13RydHe0IIwa2TAJnHjv/a6m/N\niiq/eHJZAIWKAm2zP9pZCDV1FEJx+HDy+L9L/i8q6bYs3M5l5xi7+HeFW6Hi1jff\nu7LQ4p5Ms4tsElI=\n-----END CERTIFICATE-----\n",
            "PrivateKey": "-----BEGIN RSA PRIVATE KEY-----\nMIIEowIBAAKCAQEAvD1Xlq3VFv3Wn+wFr9ZxmsXfrVS3veUQ5PgOqzpq+bqBQXTt\njin3WIN3Da97YhNl23AWtNJa3YcTBwDbzbbwfmAKZso18y0FtF6CXuh+gdvENkYk\nr5beHpNcOIeY/FDRpig2by/T4D3FXnCjKopB47KD8SjvyHXAdEmdbBJ2OpkKxUrV\nSSOkXlBQFSMSuhZ24CYYbJVNtvrxeoVr43ZY5emA8iKXrofoY8hmOqKRVL+obO6B\nyr8Z0j12/nP0Jin1n0IjrMCiYYGXBHhbzn2KzDbQj14Bd0wx2gB2ydPljwpZQ5bz\nbQshSHDfMg+SQcrj5l1SWD/loN2nP/ZsS5Pg/wIDAQABAoIBACOMzjLlx32dGOCA\n+Z34uOHLBvA8NKtHTIaBlnud/8AECg8rnwWfRVhRE7Xg80NVeIIVzCQAKir2LJDB\nB8H1D7w+NRiujbvMP+yNgL+d1u59a7P4UUtcCbzqhZsjeLAGL2Ha7FTZSoFqCRFJ\n4nbRP5paB3MPESHhoyQTFwjm/68XDzUUNman/7235ms85X6VOoSuF936JgkxMNoi\nCfcMQ9kfII6QKqDbB5LxY0P0PWc4CMgKCf492yRARFJC4HUoT/eXH+lYh+DOYlWw\nQt1Y54fxWVeG+Rz46uKsL/5+8jqCd5Ah2gfPc59Ji7DZlls2JXwRCdEvjlGFj0mO\n7qTa9tkCgYEA3zoN+ec2B6xC71vtLFGpls2yPyLnjprXkEeJQNF9yIewuYpGVGLP\ndcAKLSiB+/YvuIKZul/uBdQkHRezXJIlClWzsATZWlIcr84ZUtf4hkwgW0EmpJXJ\n3069+uTLcXr0s7mOuep1TlLEWceaiwOWZW+JYodriDsJWrRGz4G0RYsCgYEA1+BM\n5b9o0M2cMdoMqaE6XTBJRVEAFl3dokiUtQxXHEi7bDfr9Sx0Ps6pg7CdXBufDovP\nqLzfEqqnOnOVFGxaXDNhvoqLW/itn6ANPaw1XiqlhTFrFD3Rqtm194wdo5mKfILo\njgese/hb+t6RloWdaDmYBN3vaJC1cXFPWN5GiN0CgYBN89IJoOpHR6qgN7PdNC9K\n0E4cqi2+qOf6JGET15RbQLdAM79XnKHh9swW9PxfZptHjaPtZ66RLoHl/u7NtuNk\ndoUnRKo6Vk5aPliti2noTBFIjLnX4875QmApi1hYKp3lXTkwR2Xrkg+rYn7faMNO\nbOLHG487pZIgsK/BqwOu/QKBgQCnazL38uxNE0iRePPdEkb7QplwgpM4xW8/jl6V\n0o40R0vjb7M1H1a/5vKcSPqhFmLSmydfS6sNBQBQWpdBkY66drbVWQkfOMseQrhC\nHi39a8GWfG748cCLafCvnSDXYhp+2d+VVuoz8rcS5k2umM0sqY32KFClnaS56BCL\ncUbumQKBgED/8Ovu4uowO+BWbiYMX5Eov0iEk92e02SIBvCsqoe63iO7RJtdsfEr\n+QPNz35K/xyXSEjZi6gj1ui9Td91cuUC2OwRbor+yAwXgCFuxNncjQY28Ul3F4TH\n/zFuuJlb5ca/6TqMc5hxC0qdOPg3weK1dRlDfQb/vikbBD40QA/6\n-----END RSA PRIVATE KEY-----\n",
            "IssuingCA": "-----BEGIN CERTIFICATE-----\nMIICqDCCAZACCQDFLOUQOxJGoTANBgkqhkiG9w0BAQsFADAVMRMwEQYDVQQDDApm\nbG9tZXNoLmlvMCAXDTIzMDgwOTEzMzAwMloYDzMwMjIxMjEwMTMzMDAyWjAVMRMw\nEQYDVQQDDApmbG9tZXNoLmlvMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKC\nAQEAx3s/KDkaRTd1yL+Ku0qfHsj2M6JyjGycZvcB7m+rxdKqCRvZXZa7dk5qfqGE\nN7xyIQQQTWvZB+yYQPtvDnB184Idyh3h/5khfkX/V2bfaifb4v85SKpdcKIq3MsG\nQe3oBaYUeOoRHJiw9Cb2bHthtDjlsfXxm8LXf1YGHShWDZqOnGd/P7Hvw4ZQ5P+v\nBwpwpuAPgcIA3gq6qb22LZbHeENCulIfqd7giUVd6NtBP5G3Lu4ATGpwhIBnJliF\nw3VSG9G7zrH6qNx4E9qlI+PvGLidQg+qJM6J+y3dvtVICoAMD5yDOgKttmP65BYR\nQgyQFxY7zj1UwSGxGUT4waIDjQIDAQABMA0GCSqGSIb3DQEBCwUAA4IBAQADpzp7\n6nBQufE9Yhr8D9QRDT4oXo/eEO7jY274LjB/YhdZ2SN5kQXHneF91x4NsA3qTwyn\nMIRPJT9QwJuYKtX+S7lBxUh39WdtLI+Q1L2bB3DB5PEafeAJHyFszz6Gk8GcD1qu\nwinL7Fy111MiMXlU2R3rvm/z0VkDhW0vx4VaOuIgWDt/ou0jSL2xOP7aH14MZ5FG\nvIVhyVsY+O8RJj5yg9Bzso0wj4sMvNJgFEmA0ENY9KeoULcHcyfMN5fSoA5Qw/6l\nZ1Ac5sYW9UwcDzKyXtzWvabJ3lwPLnviPoorowmmT3rhhNvrVhH4jdyrLnTaFzGo\nAfYeWwziVKdGC5Yy\n-----END CERTIFICATE-----\n"
          }
        ]
      }
    }
  ],
  "RouteRules": {
    "8443": {
      "example.com": {
        "backendService1": {
          "Weight": 100
        }
      }
    }
  },
  "Services": {
    "backendService1": {
      "Endpoints": {
        "127.0.0.1:8081": {
          "Weight": 100
        },
        "127.0.0.1:8082": {
          "Weight": 100
        }
      }
    }
  },
  "Chains": {
    "TLSTerminate": [
      "common/tls-termination.js",
      "common/consumer.js",
      "tls/forward.js"
    ]
  }
}
```

多次执行下面的命令，访问负载均衡器的 `8443` 端口，可以看到轮流返回两个后端服务的响应。

```shell
curl --cacert ca-cert.pem https://example.com --connect-to example.com:443:127.0.0.1:8443
```
