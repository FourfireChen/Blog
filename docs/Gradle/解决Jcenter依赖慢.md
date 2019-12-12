# 解决Jcenter依赖慢

Android Studio，对项目的依赖有更改，需要依赖到[https://jecenter.bintray.com](https://jecenter.bintray.com)时，都非常非常慢，除非对Gradle设置代理。今天仔细找了一下，有两个更好的解决办法。

## No.1 https协议改为http协议

修改项目根目录下的`build.gradle`文件：

```groovy
buildscript {
    repositories {
        // jcenter()
        jcenter(){url 'http://jcenter.bintray.com/'}
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:2.3.0'
        
        // NOTE: Do not place your application dependencies here;they belong
        // in the individual module build.gradle files}
}
allprojects {
    repositories {
        // jcenter()
        jcenter(){url 'http://jcenter.bintray.com/'}
    }
}
```
## No.2 修改Maven仓库为国内仓库

同样修改根目录下的`build.gradle`文件：

```groovy
allprojects {
    repositories {
        // jcenter()
        // mavenCentral()
        maven { url 'http://maven.aliyun.com/nexus/content/groups/public' }
    }
```

想要一劳永逸？

直接修改Gradle的初始化脚本，也即运行时的全局配置就好了。

在`USER_HOME/.gradle/`下新建文件`init.gradle`

```groovy
allprojects{
    repositories {
        def REPOSITORY_URL = 'http://maven.aliyun.com/nexus/content/repositories/central/'
        all {
            ArtifactRepository repo ->
            if(repo instanceof MavenArtifactRepository){
                def url = repo.url.toString()
                if (url.startsWith('https://repo1.maven.org/maven2') || url.startsWith('https://jcenter.bintray.com/')) {
                project.logger.lifecycle "Repository ${repo.url} replaced by $REPOSITORY_URL."
                    remove repo
                }
            }
        }
        maven {
        	url REPOSITORY_URL
        }
}
```
