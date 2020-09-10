## 概述

弹性容器服务支持配置[ NAT 网关](https://cloud.tencent.com/document/product/215/4975)和[路由表](https://cloud.tencent.com/document/product/215/4954)实现集群内服务访问外网。

## 配置方法

1、登录[腾讯云私有网络- NAT 网关](https://console.cloud.tencent.com/vpc/nat)控制台，点击【新建】创建与EKS集群同地域、同 VPC 的 NAT 网关。
![][5]

2、登录[腾讯云私有网络-路由表](https://console.cloud.tencent.com/vpc/route)控制台，点击【新建】创建与EKS集群同地域、同 VPC 的路由表。

![][3]

3、配置路由策略：

目的端：选择要访问的外网 IP 地址，支持配置 CIDR，例如填写 0.0.0.0/0 会转发所有流量到 NAT 网关；

下一跳类型：选择 【 NAT 网关】类型；

下一跳：选择第一步创建的 NAT 网关；

![][4]

4、完成配置路由后，选择子网关联到该路由表，被选择子网内访问 Internet 的流量将指向 NAT 网关。
![][1]
5、完成路由表关联子网后，同 VPC 的资源即可以通过 NAT 网关的外网 IP 访问 Internet。

[1]:https://main.qcloudimg.com/raw/7b44c378307350f2d9c75218747bd47b.png
[3]:https://main.qcloudimg.com/raw/c9e578f4cffee808b5f5ab36e69372cf.png
[4]:https://main.qcloudimg.com/raw/b5fe2a0369befd9012f892240d22bc10.png
[5]:https://main.qcloudimg.com/raw/55da9bb3a9284d60625dd9ec1908b4ac.png
