---
title: "aliyun-api-py"
published: 2024-07-23
description: "一个简单的阿里云API封装库"
status: "archived"
tags:
  - "Python"
link:
  - label: "GitHub"
    icon: "fa7-brands:github"
    value: "https://github.com/AkariRin/aliyun-api-py"
  - label: "PyPI"
    icon: "material-symbols:package-2"
    value: "https://pypi.org/project/aliyun-api-py/"
---

一个简单的阿里云API Python封装库
## 使用API封装库
```python
import aliyun_api_py

# 创建API请求
request = aliyun_api_py.Api(access_key_id, access_key_secret, http_method, host, uri, x_acs_action, x_acs_version,algorithm)
request.param["your-param"] = "sth"
request.body["your-body"] = "sth"

try:
    print(request.exec())
except ConnectionError as e:
    print(e)
```
`access_key_id`：阿里云AccessKey ID

`access_key_secret`：阿里云AccessKey Secret

`http_method`：http请求方式，例如`POST`

`host`：服务地址，详见[服务区域列表](https://api.aliyun.com/product/Ecs)

`uri`：资源路径

`x_acs_action`：API名称

`x_acs_version`：API版本

`algorithm`：签名算法，默认为`ACS3-HMAC-SHA256`，一般情况下无需更改

更多参数信息详见[请求结构和签名机制](https://help.aliyun.com/zh/sdk/product-overview/v3-request-structure-and-signature)
### 返回格式
当请求正确发出时，返回格式如下：
```json
{
  "status_code": "状态码",
  "headers": "响应头",
  "body": "返回的内容，参见对应API文档"
}
```
若请求出错（不包括HTTPError），则会抛出异常
## 使用SDK
**目前SDK随缘更新，碰到自己需要的API可能会随手写个SDK方便调用**

API相关信息详见[阿里云API文档](https://api.aliyun.com/document)
```python
import aliyun_api_py

# 调用请求函数（以重启ECS为例）
request = aliyun_api_py.Ecs(access_key_id, access_key_secret, host)
print(request.reboot_instance(instance_id, force_stop, dry_run))
```
### ECS
`ecs.reboot_instance(instance_id, force_stop, dry_run)`：重启ECS实例
