# Android库发布到Maven Central 流程：

# 第一步·注册
Maven Central是由sonatype运营的，那么首先需要去注册一个sonatype的账号并获得仓库使用许可。

先前往issues.sonatype.org注册账号，界面如图，点击红框部分：  
![注册](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e0f5b0887fa4480ebad6f055c11a70d9~tplv-k3u1fbpfcp-zoom-1.image)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/66e5dde710184d528454a71be6511c38~tplv-k3u1fbpfcp-zoom-1.image)

名称直接填写项目名就行，重点是Group ID,一般填写github的地址:io.github.xxx,然后等待官方回复，官方会让你创建一个repo来验证（下图），按照他说的来就行，这一步有的同学说很慢，但是我这边还是挺快的

![](https://github.com/cwsky0221/notes/blob/main/pic/3FA44E50-5911-46BC-A8DC-181F6972A922.png?raw=true)

# 第二步·Gradle准备
接下来需要进入你的项目，完成一系列的配置，请先参考下载下边的gradle文件放置在项目的根目录，请注意文中中文部分请按照需要自行修改：

文件：publish-mavencentral.gradle

```
apply plugin: 'maven-publish'
apply plugin: 'signing'

task androidSourcesJar(type: Jar) {
    classifier = 'sources'
    from android.sourceSets.main.java.source
}

ext["signing.keyId"] = ''
ext["signing.password"] = ''
ext["signing.secretKeyRingFile"] = ''
ext["ossrhUsername"] = ''
ext["ossrhPassword"] = ''

File secretPropsFile = project.rootProject.file('local.properties')
if (secretPropsFile.exists()) {
    println "Found secret props file, loading props"
    Properties p = new Properties()
    p.load(new FileInputStream(secretPropsFile))
    p.each { name, value ->
        ext[name] = value
    }
} else {
    println "No props file, loading env vars"
}
publishing {
    publications {
        release(MavenPublication) {
            // The coordinates of the library, being set from variables that
            // we'll set up in a moment
            groupId PUBLISH_GROUP_ID
            artifactId PUBLISH_ARTIFACT_ID
            version PUBLISH_VERSION

            // Two artifacts, the `aar` and the sources
            artifact("$buildDir/outputs/aar/${project.getName()}-release.aar")
            artifact androidSourcesJar

            // Self-explanatory metadata for the most part
            pom {
                name = PUBLISH_ARTIFACT_ID
                description = '你的项目描述'
                // If your project has a dedicated site, use its URL here
                url = 'Github地址'
                licenses {
                    license {
                    	//协议类型，一般默认Apache License2.0的话不用改：
                        name = 'The Apache License, Version 2.0'
                        url = 'http://www.apache.org/licenses/LICENSE-2.0.txt'
                    }
                }
                developers {
                    developer {
                        id = '用户ID'
                        name = '用户名'
                        email = '邮箱'
                    }
                }
                // Version control info, if you're using GitHub, follow the format as seen here
                scm {
                	//修改成你的Git地址：
                    connection = 'scm:git:github.com/xxx/xxxx.git'
                    developerConnection = 'scm:git:ssh://github.com/xxx/xxxx.git'
                    //分支地址：
                    url = 'https://github.com/xxx/xxxx/tree/master'
                }
                // A slightly hacky fix so that your POM will include any transitive dependencies
                // that your library builds upon
                withXml {
                    def dependenciesNode = asNode().appendNode('dependencies')

                    project.configurations.implementation.allDependencies.each {
                        def dependencyNode = dependenciesNode.appendNode('dependency')
                        dependencyNode.appendNode('groupId', it.group)
                        dependencyNode.appendNode('artifactId', it.name)
                        dependencyNode.appendNode('version', it.version)
                    }
                }
            }
        }
    }
    repositories {
        // The repository to publish to, Sonatype/MavenCentral
        maven {
            // This is an arbitrary name, you may also use "mavencentral" or
            // any other name that's descriptive for you
            name = "项目名称"

            def releasesRepoUrl = "https://oss.sonatype.org/service/local/staging/deploy/maven2/"
            def snapshotsRepoUrl = "https://oss.sonatype.org/content/repositories/snapshots/"
            // You only need this if you want to publish snapshots, otherwise just set the URL
            // to the release repo directly
            url = version.endsWith('SNAPSHOT') ? snapshotsRepoUrl : releasesRepoUrl

            // The username and password we've fetched earlier
            credentials {
                username ossrhUsername
                password ossrhPassword
            }
        }
    }
}
signing {
    sign publishing.publications
}

```

# 第三步·签名秘钥申请

我这边使用的mac，就只说mac的步骤了，windows也差不多
下载地址：https://www.gnupg.org/download/index.html   


![秘钥](https://github.com/cwsky0221/notes/blob/main/pic/6FA21D24-6757-4E70-ADD0-03FF7F75EBD3.png?raw=true)

下载之后直接生成key,并且需要点击在服务器上发布，网上都说直接导出key成gpg文件，但是我这边导出的文件后面不能识别成功，我采用的是命令的方式：
```
gpg --export-secret-keys -o secring.gpg

```
回到Android Studio，到local.properties文件中，编辑刚刚的文件：
```
signing.keyId=02C4AC4B //keyId 是生成key的指纹的后8位
signing.password=你的秘钥密码
signing.secretKeyRingFile=\xxx\你的GPG签名秘钥文件.gpg
ossrhUsername=sonatype帐号
ossrhPassword=sonatype密码
```

# 第四步·上传
先进入你的Module的build.gradle，添加以下代码：
```
ext {
    PUBLISH_GROUP_ID = "com.xxx.xxxx"		//项目包名
    PUBLISH_ARTIFACT_ID = 'xxxx'			//项目名
    PUBLISH_VERSION = 0.0.1					//版本号
}
apply from: "${rootProject.projectDir}/publish-mavencentral.gradle"

```

Sync后，先通过Gradle编译release版本，双击Module名称/Tasks/build/assemble,完成后，上传到Maven Central，双击Module名称/Tasks/publishing/publicReleasePublicationToXXXRepository

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d0700693f8314302a4f9f6fc6e5f9463~tplv-k3u1fbpfcp-zoom-1.image)
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/090cad7fcf1549e89a77a52b145e6a7b~tplv-k3u1fbpfcp-zoom-1.image)

执行完毕后，进入oss.sonatype.org/ ，使用你的sonatype帐号和密码登录。

进入左侧的Staging Repositories
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d43edd8abef5438d807070ace4255e20~tplv-k3u1fbpfcp-zoom-1.image)

在右上角搜索你的名字，应该能找到你上传的库文件，点击下方的Content可以查看详细已上传的文件：
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b9fe284feeea416aaae0236281672c36~tplv-k3u1fbpfcp-zoom-1.image)

确认无误后，点击Close关闭，输入介绍后点击Confirm完成所有操作。
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/964d125711d2411dabf9621a4ebd15aa~tplv-k3u1fbpfcp-zoom-1.image)

接下来，需要等待5-10分钟，待Activity选项卡中出现对号的“Repository closed”即完成准备，然后点击Release按钮发布到MavenCentral，等待几个小时后可以在 search.maven.org/ 查询发布结果。
