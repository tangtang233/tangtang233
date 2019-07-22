---
title: Hexo多服务器一键部署
date: 2019-07-04 17:07:11
categories: 程序员笔记
tags: 程序员笔记
---
## 自动化部署成果和测试

### 由头

这两天看了一些自动化部署的方案，就想着把自己公司的部署方式搞成自动化的。其实还是自己懒，实在不想每次都打好包再远程服务器去替换、重启。

这里得说一下前提，因为历史原因，公司服务器用的阿里云，且是Windows系统（好烦啊，为啥不买Linux的）。
项目结构也简单，后端就一个，SpringBoot框架的，打包出来是个jar，我在服务器上写个bat文件直接java -jar就完事儿。
前端有好几个东西，包含微信的H5、web端的管理平台、公司的宣传网站。
因为业务量也不大，所以用不着分布式啥的。就直接一个nginx映射几个成几个子域名，后端接口也通过nginx映射后供前端页面调用。

总结一下现在更新方式：
    1. 后端更新：直接本地打jar包->远程服务器->停服务->备份旧jar包->拷贝新jar包->起服务
    2. 前端更新：前端妹子打好包（Vue开发的，就是个dist.rar）发我->本地解压改名->远程服务器->备份旧的前端文件->替换成新的

### 实现方式

业务比较简单，也用不着啥Jenkins这些持续集成啥的东西了，就Git Hook应该满足需求。
我现在在自己的阿里云服务器（CentOS）上试Git Hook，等成功了再搞公司服务器去。
先写到这边。

### 结果

折腾了好几天，最终还是优先对自己的Java后端项目做了自动化部署的处理。下面总结几点：

- 使用的是Gradle，所以选择了gradle-ssh-plugin插件来辅助完成
- 公司服务器是Windows平台，所以需要额外安装ssh服务

下面给出我的gradle-ssh-plugin这个插件的使用示例：

1. 引入插件
   在build.gradle中，引入插件

   ```gradle
    buildscript {
    ext {
        springBootVersion = '2.1.1.RELEASE'
        sshPluginVersion = '2.10.1'
    }
    repositories {
        maven {
            url 'http://maven.aliyun.com/nexus/content/groups/public/'
        }
        mavenCentral()
    }
    dependencies {
        classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
        classpath("org.hidetake:gradle-ssh-plugin:${sshPluginVersion}")
        }
    }


    apply plugin: 'org.hidetake.ssh'```

2. 配置远程地址信息

    仍然是在build.gradle中编写：

    ```gradle
    ssh.settings {
        //支持多服务器部署（一般用于集群）
        knownHosts = allowAnyHosts
    }

    //远程服务器信息
    remotes {
        //这个deployServer名字是自定义的，可以配置多台服务器
        deployServer {
            host = '192.168.0.163'
            user = 'Administrator'
            // 支持password、rsa等方式登录
            password = '123456'
            //identity = file("${rootDir}/id_rsa")
        }
        testServer{
            host = '192.168.0.165'
            user = 'root'
            // 支持password、rsa等方式登录
            password = '123456'
            //identity = file("${rootDir}/id_rsa")
        }
    }


3. Gradle任务编写

    最后就是编写你的部署逻辑啦。这个会因为每个项目技术栈的不同，服务器的环境不同，可能会有各种细节上的出入，但逻辑还是可以参考的。当然，也可以Google一下跟自己累死的部署方案是什么逻辑。
    一般部署逻辑无外乎：停止服务->备份上一版本->传输新部署包->重启服务，如果是tomcat下部署，那可能涉及解压等等其他操作，这些都可以通过shell命令去完成，这里就不赘述了。我这里得shell命令示例基于我自己的服务器情况，服务器环境是Windows，已经安装了Cygwin（让Windows平台下可以运行shell命令，支持ssh插件），项目是SpringBoot项目，Gradle构建，直接在开发机器上打成jar包，在服务器上直接使用java -jar xxxx.jar命令来启动。

```gradle

    // 定义一个获取当前时间字符串的辅助函数
    static def now() {
        new Date().format('yyyyMMddHHmmss')
    }

    // 定一个pkg任务
    task pkg
    pkg.dependsOn 'clean'
    pkg.dependsOn 'bootJar'
    bootJar.shouldRunAfter 'clean'
    // 定义自动部署脚本
    task deploy(dependsOn: pkg) {
        doLast {
            println("start task deploy:${now()}")
            ssh.run {
                session(remotes.deployServer) {
                    // 停止原有服务
                    execute '/cygdrive/c/winsw/stop.bat'
                    // 备份原有程序，首次部署mv会执行失败，忽略错误
                    execute "mv /cygdrive/c/winsw/saas.jar /cygdrive/c/winsw/backup/saas.${now()}", ignoreError: true
                    // 上传新版本程序
                    put from: 'build/libs/freelycar_saas-' + version + '.jar', into: '/cygdrive/c/winsw/saas.jar'
                    // 赋予可执行权限
    //            execute 'chmod u+x /cygdrive/c/winsw/saas.jar'
                    // 启动新服务
                    execute '/cygdrive/c/winsw/start.bat'
                }
            }
        }
    }```

这样gradle中就多了两个自定义任务，名字分别为pkg和deploy的task任务，并且deploy依赖于pkg的成功。
pkg任务其实就是执行了gradle自带的clean（清理）和bootJar（打jar包）任务，成功后执行deploy任务。


以上就是我的自动化部署的一种实现方式。其实这段时间也是稍微试了一下Jenkins，后面如果用起来了再总结一下Jenkins的使用感受吧。
如果有疑问，欢迎大家留言，我会不定期回复的。
