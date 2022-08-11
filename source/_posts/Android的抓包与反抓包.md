---

layout: android
title: Android抓包与反抓包
date: 2022-07-21 15:39:51
tags:
---

### “裸奔”的APP

针对未启用Https、也做任何防护的App，直接使用抓包工具如[Charles](https://www.charlesproxy.com/download/latest-release/)、[Fiddler](https://link.juejin.cn/?target=https%3A%2F%2Fwww.telerik.com%2Fdownload%2Ffiddler)即可抓取到数据。

### 使用了HTTPS

Google在7.0更新了安全策略，APP 默认不信任用户域的证书。用户自己安装的证书便不会被信任。所以在不同的Android的攻击方案不同。

#### Android7.0以下

直接将抓包工具的根证书安装至Android系统中，即可抓取。

#### Android7.0以上

##### 将证书安装到系统目录下

root后将证书安装到`/system/etc/security/cacerts/`目录下

##### 修改app的xml源码

可以通过反编译后在`AndroidManifest.xml`中增加

```
<?xml version="1.0" encoding="utf-8"?>
    <manifest ... >
    	<application android:networkSecurityConfig="@xml/network_security_config"
        ... >
        ...
    </application>
</manifest>
```
并创建`res/xml/network_security_config.xml`文件

```
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <domain-config>
        <domain includeSubdomains="true">你要抓取的域名</domain>
        <trust-anchors>
        	<certificates src="user"/>
        </trust-anchors>
    </domain-config>
</network-security-config>
```







### 正确的使用Https

在构建`HostnameVerifier`时，需要比对hostname是否和证书中的域名信息是否一致

```
HostnameVerifier hostnameVerifier = new HostnameVerifier() {
    @Override
    public boolean verify(String hostname, SSLSession session) {
        return hostname.equals(session.getPeerHost());
    }
};
```










1. 校验自身应用签名（防止重打包）

2. 对服务器证书域名进行强校验

3. 自身内置证书，校验CA证书

   1. 证书锁定：比对证书
   2. 密钥锁定：比对密钥

4. 代理检测

5. 检测手机是否安装SSL攻击工具（Xposed+JustTrustMe&SSLkiller ）

   1. 在native层实现，增加逆向难度

6. 服务器与客户端双向验证

   ![img](https://image.3001.net/images/20210911/1631371570_613cc13233d55da892c11.png)

   > 但这种双向认证开销较大，且安全性与SSL pinning一致，目前大多数app都采用SSL Pinning这种方案。

7. 传输内容加密

8. 使用修改过的ssl库（抖音）



https://bbs.pediy.com/thread-269028.htm