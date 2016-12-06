##tomcat配置ssl证书

####修改server.xml配置文件
######1.打开ssl配置段,开启8443端口
```

    <!--
    <Connector port="8443" protocol="org.apache.coyote.http11.Http11Protocol"
               maxThreads="150" SSLEnabled="true" scheme="https" secure="true"
               clientAuth="false" sslProtocol="TLS" />
    -->
```
修改成
```
    <Connector port="8443" protocol="org.apache.coyote.http11.Http11Protocol"
               maxThreads="150" SSLEnabled="true" scheme="https" secure="true"
               keystoreFile="conf/test.jks" 	#keystoreFile配置证书,证书需要事先准备好
                keystorePass="12345678"		#证书密码
                keystoreType="JKS" 	#证书类型

               clientAuth="false" sslProtocol="TLS" />
```


######3.修改主端口配置段,添加redirectPort="443"
```
<Connector port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               acceptCount="1500"
               enableLookups="false"
               URIEncoding="UTF-8"
               redirectPort="8443" 	#添加的参数
               useBodyEncodingForURI="true"/>
```


###配置http跳转https
######修改web.xml,添加如下配置：
```
<login-config>
        <!-- Authorization setting for SSL -->
        <auth-method>CLIENT-CERT</auth-method>
        <realm-name>Client Cert Users-only Area</realm-name>
</login-config>
<security-constraint>
        <!-- Authorization setting for SSL -->
        <web-resource-collection >
            <web-resource-name >SSL</web-resource-name>
            <url-pattern>/*</url-pattern>
        </web-resource-collection>
        <user-data-constraint>
            <transport-guarantee>CONFIDENTIAL</transport-guarantee>
        </user-data-constraint>
</security-constraint>
```


####自签名证书的创建方法
#tomcat自签名证书
```
/usr/java/jdk1.7.0_51/bin/keytool -genkey -alias mydomain -keyalg RSA -keystore keystore.jks -keysize 2048
Enter keystore password:  
Re-enter new password: 
What is your first and last name?
  [Unknown]:  jiuyun    #您的名字与姓氏是什么?
What is the name of your organizational unit?
  [Unknown]:  jiuding     #您的组织单位名称是什么?
What is the name of your organization?
  [Unknown]:  jiuding     #您的组织名称是什么?
What is the name of your City or Locality?
  [Unknown]:  Hangzhou    #您所在的城市或区域名称是什么?
What is the name of your State or Province?
  [Unknown]:  Zhejiang    #您所在的省/市/自治区名称是什么?
What is the two-letter country code for this unit?
  [Unknown]:  China     #该单位的双字母国家/地区代码是什么?
Is CN=jiuyun, OU=jiuding, O=jiuding, L=Hangzhou, ST=Zhejiang, C=China correct?
  [no]:  yes

Enter key password for <mydomain>
        (RETURN if same as keystore password):      #输入证书密码
Re-enter new password: 
```