# Jenkins Library

## 需要的插件

1. Groovy PostBuild
2. build-user-vars-plugin
3. HTTP Request
4. AnsiColor

## 一般使用

```groovy
@Library('apzda') _

pipeline {
    agent any

    options {
        // 启用颜色支持
        ansiColor('xterm')
        //设置在项目打印日志时带上对应时间
        timestamps()
        // 设置流水线运行超过n分钟，Jenkins将中止流水线
        timeout time: 60, unit: 'MINUTES'
        // 禁止并行构建
        disableConcurrentBuilds()
        // 表示保留50次构建历史
        buildDiscarder(logRotator(numToKeepStr: '50', artifactNumToKeepStr: '0'))
    }
    environment {
        SERVICE_NAME = "service-name"
        SERVICE_ENV = "UAT"
        SERVICE_VER = "1.0.0-SNAPSHOT"
    }

    stages {
        stage('scm') {
            steps {
                pullcode branch: "ahaha", credentialsId: "abc", url: "https://asdfadsf.com/project.git"
            }
        }
        stage("套用Assembly模板") {
            steps {
                assembly module: "${SERVICE_NAME}", layerjar: true, docker: true, assembly: true
            }
        }
        stage('Build') {
            steps {
                sh 'mvn -T 8C package -P prod'
            }
        }
        stage('套用镜像模板 - layerjar') {
            steps {
                dockertpl tpl: 'layerjar', module: "${SERVICE_NAME}", jdkImage: 'openjdk:17'
            }
        }
        stage("Docker build") {
            steps {
                sh "docker build -t apzda/${SERVICE_NAME}:${BUILD_DATE}-${BUILD_ID} --build_arg SERVICE_NAME=$SERVICE_NAME --build-arg SERVICE_VER=$SERVICE_VER --build-arg SERVICE_JAR=$SERVICE_NAME-$SERVICE_VER.jar --compress ./$SERVICE_NAME/target/docker"
            }
        }
        stage('套用镜像模板 - nginx') {
            steps {
                dockertpl tpl: 'nginx'
                sh "docker build -t apzda/nginx:${BUILD_DATE}-${BUILD_ID} --compress --no-cache ."
            }
        }
    }

    post {
        always {
            wechat token: ''
        }
    }
}
```

## VARS

1. `pullcode`: 拉取代码
    - `branch`: 分支
    - `url`： 仓库地址
    - `credentialsId`: git 用户凭证
2. `wechat`: 发送企业微信通知
    - `token`: Token， 为空或`false`时将不会发送通知
    - `type`: 消息类型（**可选**）
    - `message`: 消息（**可选**）
3. `dockertpl`: 基于模板生成Dockerfile, docker-entrypoint.sh, env.sh等文件
    - `tpl`: nginx, openresty, fatjar, layerjar, jar, php
    - `basedir`: type = nginx 或 openresty时指定,默认为"/"
    - `module`: maven的子模块,不指定时默认为当前模块
    - `jdkImage`: jdk镜像名, java必须在路径上，默认使用openjdk:17
    - `nginxImage`: nginx镜像名, 默认使用nginx:1.25.0-alpine
    - `restyImage`: openresty镜像名, 默认使用bitnami/openresty:1.21.4-1
    - `path`: Dockerfile等文件存放位置,默认为"."或"${module}/target/docker"
    - `config`: nginx 或 openresty 的配置文件路径,不指定时使用默认配置
4. `assembly`:
    - `module`: maven的子模块,不指定时默认为当前模块
    - `assembly`: [true|'force'] assembly-descriptor.xml
    - `docker`: [true|'force'] assembly-docker.xml
    - `layerjar`: [true|'force] layers.xml
    - `logback`: [true|'force] 应用logback-spring.xml模板
    - `skywalking`: [true|false] 使用支持skywalking的logback配置文件

> 特别注:

1. 当使用`dockertpl`生成Dockerfile时, java项目在构建镜像时，需要指定以下构建参数:
    * `SERVICE_JAR`: jar文件，必须在`path`参数指定的目录中
    * `SERVICE_NAME`: 应用名（服务名）
    * `SERVICE_VER`： 版本
2. 当使用`nginx`和`openresty`模板时，`dist`目录必须在`path`指定的目录里.
