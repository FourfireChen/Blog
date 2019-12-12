# Android Studio Gradle代理设置

Gradle代理和Android Studio的Setting中设置的代理不同，在Setting中设置的代理，Gradle是没用的，Gradle必须配置才能使用代理。

代理一般有两种，一种是socks代理，一种是http代理，gradle针对这两种代理的设置方式不同。

### http

编辑gradle.properties文件
```groovy
#systemProp.socks.proxyHost=127.0.0.1
#systemProp.socks.proxyPort=1080

#systemProp.https.proxyHost=127.0.0.1
#systemProp.https.proxyPort=1080

#systemProp.https.proxyHost=socks5://127.0.0.1
#systemProp.https.proxyPort=1080
```
### Socks

编辑gradle.properties文件

```groovy
org.gradle.jvmargs=-DsocksProxyHost=127.0.0.1 -DsocksProxyPort=1080
```

### 坑

个人第一次新建kotlin项目的时候，下载kotlin-compiler没挂代理真的不行，但是之后正常情况下打开，挂了代理真的不行！！

观察了一下，是下载https://jcenter.bintray.com时就非常慢！！！
