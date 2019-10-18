---
title: jenkins
date: 2019-10-17 22:38:38
tags: [jenkins]
---
# Jenkins踩坑

最近弄了下`jenkins`，就是为了部署方便点，然后也遇到一些麻烦，顺便记录下

## 安装Jenkins
这个直接安装再本地机器上，要不就在`Docker`里面装，`Docker`里面装的话就会有打包时候无法使用`docker`命令的问题，这个需要解决下，其他的都一样。直接装最简单的就是`apt install jenkins`

然后进入之后插件什么的装一下就妥了，安装插件之前可以先升级下站点的地址，也就是插件的地址，自带那个有点慢，例如这个`https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/current/update-center.json`，在`插件管理/高级/升级站点`贴上就行了

额外要安装的插件：

1. `Publish Over SSH`，用于远程登录机器用的
2. `Generic Webhook Trigger`，用来做触发器的

其他的点点就行了

## 任务

新建任务的时候，可以选择`自由风格的软件`或者`pipeline`，后面分别说下，这两个还是有点区别的

### 自由风格
这个用一个`html`项目做例子，完了就是一个纯静态的文件，然后发送到对应的服务器对应的位置就行了

1. **源码管理**：填写上项目的地址即可，GitHub项目https地址可以不用秘钥就能直接访问，当然我们构建只需要可读即可，所以这样最简单，例如`https://github.com/xxx/jenkins-html.git`

2. **构建环境**：这一步可以添加一个触发器，其他的参数之类的都可以不填，按需填上即可，但是要填好`token`，这是一个随意的字符串，然后在github项目的`settings`的`Webhooks`增加一个`webohook`，在`Payload URL`中填写`http://JENKINS_URL/generic-webhook-trigger/invoke?token=?`，具体的替换下就行了；如果不用触发器这一步可以什么都不做
3. **构建**：增加一个`shell`执行即可，然后写上构建的时候需要的一些命令即可，例如：

	```sh
	echo $PATH
	node -v
	npm -v
	npm install
	npm run build
	rm -rf testhtml.tar.gz
	tar -zcvf testhtml.tar.gz dist index.html
	```
4. **构建后操作**：选择`Send build artifacts over SSH`，在这之前需要先在`系统设置`中的`Publish over SSH`配置下`SSH Server`，这个配置只要填写远端服务器的`host`， 登录用户，文件发送的远端位置，要确保这个文件夹存在，然后勾选`Use password authentication, or use a different key`，密码或者私钥填一个。再到`SSH Publishers`中选上刚才配置的服务器名称，`source files`填一个要传输的文件，例如`*.tar.gz`，`Exec command`填上在远端服务器要执行的命令。

在这些都好了之后执行构建即可，稳妥~

### Pipeline
流水线的构建例子也是上面那个项目，只不过把构建过程写在文件中，这样流程也是全自己控制了

**流水线**：这里选择`Pipeline script from SCM`，这样可以在项目中写`Jenkinsfile`配置，`脚本路径`可以指定脚本名称，默认就是`Jenkinsfile`.`SCM`中选择`GIT`之后填上`github`项目地址`https://github.com/xxx/jenkins-html.git`，然后就结束了

这里重点说下`Jenkinsfile`写法，直接贴出来：

```sh
pipeline{
	agent any

	parameters {
		string(name: 'ZIPNAME', defaultValue: 'test', description: 'zip package name')
	}

	stages {
		stage('Prepare'){
			steps{
				sh 'node -v'
				sh 'npm -v'
			}
		}
		stage('Build'){
			steps {
				sh 'npm install'
				sh 'npm run build'
			}
		}
		stage('Zip'){
			steps{
				sh "rm -rf ${ZIPNAME}.tar.gz"
				sh 'tar -zcvf ${ZIPNAME}.tar.gz dist index.html'
			}
		}
		stage('Deploy'){
			steps{
				sh 'chmod +x deploy.sh'
				timeout(time: 3, unit: 'MINUTES') {
					retry(3) {
						sh "./deploy.sh ${ZIPNAME}"
					}
				}
			}
		}
	}

	post {
		always {
			echo 'I will always say Hello again!'
		}
	}
}
```
	
项目中还有`deploy.sh`脚本，这个是部署用的，主要是传输打包文件到目标机器上然后进行部署操作

这个文件可能是没有执行权限，所以执行下`chmod +x deploy.sh`

`deploy.sh`内容如下：

```sh
#!/bin/bash

scp "$1.tar.gz" username@host:/root/html/
ssh username@host << EOF
cd /usr/share/nginx/html 
whoami
hostname
rm -rf test && echo 'remove test'
mkdir test && echo 'mkdir test'
tar -zxvf "/root/html/$1.tar.gz" -C ./test
EOF
```

**重点**：

1. `ssh`之后的代码都要用`EOF`包住，说明是在远端机器登录后要执行的命令，否则都会在本地执行

2. 如果执行出现`Host key verification failed.`，需要执行下面的一波操作，因为`jenkins`运行的时候用的是`jenkins`这个用户....

	```sh
	su jenkins
	ssh-keygen
	//将公钥放到目标机器的~/.ssh/authorized_keys
	ssh-copy-id username@host
	//需要输入密码
	//或者直接copy
	```
这个特别坑...

## 最后

如果打包`Docker`镜像的话，更加简单，项目中写好`Dockerfile`即可，然后写一点镜像`build`的命令即可，可以直接往服务器发送镜像，也可以发送到`Dockerhub`之类的镜像托管平台，再通知服务器拉取镜像运行