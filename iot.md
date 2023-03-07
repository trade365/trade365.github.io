本文介绍两种常见的物联网开发平台，腾讯云IOT平台以及苹果MDM
* 腾讯云IOT平台：通过IOT平台，实现对设备的控制，管理。比如想通过API控制一个电源，灯，空调等等。
* 苹果MDM：MDM的全称是Mobile Device Management，是苹果公司为了管控苹果设备推出的一个设备管理平台。比如控制ipad安装软件/壁纸，查看电量等等。

# 腾讯云IOT
## 准备
网关
子设备（如电源控制器，灯，门禁等）
IOT平台
配网小程序

网关和子设备一般是由适配了腾讯云协议的厂商提供，IOT平台由腾讯提供，业务只需要开发配网小程序（腾讯云提供了SDK，开发非常方便）。

## 结构
网关和设备通过局域网协议进行通信，网关和IOT平台通过MQTT协议进行通信。只需要配置网关的网络，不需要配置设备的网络。
![在这里插入图片描述](https://img-blog.csdnimg.cn/a92bebfd1e90412b92e3de014866074e.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ppYW5waW5nemp1,size_16,color_FFFFFF,t_70#pic_center)
腾讯云IOT平台相关概念：
* 腾讯云账号：IOT平台管理账号
* IOT实例：管理项目和应用
* 项目：产品必须建立在项目下面
* 应用：腾讯连连应用，会有appID，appKey等，用户设备的管理
* 产品：腾讯云IOT平台上需要建立如网关，子设备的产品
* 产品物模型：物模型定义了设备的属性和行为，用来管控设备。
* 腾讯连连用户和家庭：连连是配网和管理设备用的，连连用户通过oauth2协议进行鉴权，一个用户下面有若干家庭，创建新用户时会创建一个默认的家庭。设备和网关都是在某个家庭下面管理的，如果想控制网关或者网关下的设备，都需要带上家庭ID。


## 网关配网
腾讯云提供了若干种配置网关网络的方法，下面是通过softAP方式对网关进行配网的流程：
![在这里插入图片描述](https://img-blog.csdnimg.cn/50cd1be753474aa08d45ffb407a08cb2.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ppYW5waW5nemp1,size_16,color_FFFFFF,t_70#pic_center)

![在这里插入图片描述](https://img-blog.csdnimg.cn/e633f66ad130447d8dc4bff730c5d1cc.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAamlhbnBpbmd6anU=,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)

## 子设备添加到网关
子设备添加到网关需要设备商提供相关的功能，涉及到了网关和子设备局域网的通信。步骤如下：
* 调用网关搜索网关附近的子设备
* 子设备定位：使子设备发出声音，从而知道当前添加的是哪个设备
* 添加子设备：将一个子设备添加连连，并绑定到一个网关

# 苹果MDM
需要用户自己搭建MDM服务器，推荐使用开源的：https://github.com/micromdm/micromdm

## 概念
* MDM server：业务自己维护的MDM服务
* ABM：Apple Business Manager，苹果提供的，在这里可以管理苹果设备
* APNS：Apple Push Notification service，苹果推送服务，用于和苹果设备直接通信
* 设备：如IPad，MacBook
![在这里插入图片描述](https://img-blog.csdnimg.cn/afb6f2d8b8a84232ad585c5b747d92d9.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ppYW5waW5nemp1,size_16,color_FFFFFF,t_70#pic_center)

如上图所示，MDM server需要APNS和ABM的鉴权，需要配置相关的证书，流程比较长，参照micromdm的文档配置鉴权：
https://github.com/micromdm/micromdm/blob/main/docs/user-guide/quickstart.md

MDM server和APNS之间通信遵守苹果制定的协议:
https://developer.apple.com/business/documentation/MDM-Protocol-Reference.pdf

## 设备添加
添加流程如下：
* 在ABM上添加设备，并指定mdm server
* ABM把设备同步到mdm server
* 生成dep profile：参考 https://github.com/micromdm/micromdm/blob/main/docs/user-guide/enrolling-devices.md，生成dep profile，这个dep profile是一个json格式的文件，配置了mdm server的地址等，设备安装了dep profile，才能enroll到mdm server
* 给设备指定dep profile
* IPad重新激活，激活过程中会让用户选择是否加入到MDM控制
* IPad下载dep profile，enroll到mdm server
* 至此IPad录入到mdm server。

正确录入的设备通过`./mdmctl  get devices`命令可以查看设备的EnrollmentStatus状态为true。

成功录入的设备就可以HTTP调用`/v1/commands`进行设备控制。

设备控制的流程可以参考上图：
* Mdm通知APNS有新的控制指令
* APNS来mdm取指令
* APNS控制设备
* APNS调用mdm返回结果

如果设备不在线，APNS会把指令存起来，等设备在线了，会再次控制设备，再调用mdm返回结果。
