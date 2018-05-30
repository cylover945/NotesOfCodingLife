# SpringBoot配置HTTPS
## 生成证书
使用HTTPS需要为服务配置一个证书，证书可以自己生成的自签名证书，也可以从ssl证书授权中心获得

接下来我使用JRE中自带的Keytool工具生成一个自签名证书。

	keytool -genkey -alias tomcat -storetype PKCS12 -keyalg RSA -keysize 2048 -keystore keystore.p12 -validity 3650

## 配置证书
1. 将证书放在resources目录下
2. 在application.yml中加入如下配置

		server:
		  port: 8443
		  ssl:
		    key-store: classpath:keystore.p12
		    key-store-type: PKCS12
		    key-alias: tomcat
		    key-store-password: 123456

## 结果查看