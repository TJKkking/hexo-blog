---
title: 搭建Gitlab与本地镜像仓库
date: 2023-11-23 10:27:15
tags:
---

主要分为两步，Gitlab安装和镜像仓库配置。
# Gitlab安装
## 前置准备
为了灵活与安全，选择采用Docker安装，首先得安装Docker。
查看Docker版本：
```shell
docker version
```
Gitlab容器的配置、数据、日志挂载在宿主机的`/srv/gitlab`目录下。
首先设置个路径，在`~/.bash_profile`或者`~/.bashrc`中添加：
```shell
export GITLAB_HOME=/srv/gitlab
```
设置好记得source一下：
```shell
source ~/.bash_profile
# 或者
source ~/.bashrc
```
## 使用 Docker Engine 安装极狐GitLab
安装命令：
```shell
sudo docker run --detach \
  --hostname gitlab.virtualplane.com \
  --publish 443:443 --publish 80:80 --publish 23:22 \
  --name gitlab \
  --restart always \
  --volume $GITLAB_HOME/config:/etc/gitlab \
  --volume $GITLAB_HOME/logs:/var/log/gitlab \
  --volume $GITLAB_HOME/data:/var/opt/gitlab \
  --shm-size 256m \
  registry.gitlab.cn/omnibus/gitlab-jh:latest
```
**有几点需要注意：**

- `--hostname`为Gitlab 的访问域名，由于不需要公网解析，在内网环境下可自己设一个方便的域名，只需要手动添加DNS解析记录或者hosts保证能正常映射ip即可。
- 端口映射。设置中将容器的`443`和`80`端口分别映射到宿主机相应端口上，但是将用于SSH的容器22端口映射到23端口，这是因为宿主机的22端口要留作宿主机的SSH连接用。
- `--volume`。`$GITLAB_HOME/config`挂载到容器的`/etc/gitlab`路径下，所以后续进行配置时的配置文件在宿主机的`$GITLAB_HOME/config`（即绝对路径`/srv/gitlab/config`）路径下能找到。
- 镜像tag一般不推荐`latest`.要使用特定的标记版本，请将 registry.gitlab.cn/omnibus/gitlab-jh:latest 替换为要运行的极狐GitLab 版本，例如 registry.gitlab.cn/omnibus/gitlab-jh:16.1.0。

配置完毕运行安装即可，为了方便排错，建议将上面的运行命令保存方便查看。
初始化过程可能需要很长时间。 可以通过以下命令跟踪此过程：
```shell
sudo docker logs -f gitlab
```
容器启动并添加好DNS解析后即可通过域名`gitlab.virtualplane.com`登录Github
![image.png](../asset/gitlabimagehost/1692839519023-d664c880-1b53-4c23-ab99-e879ca4e8402.png#averageHue=%23fefefe&clientId=uc40d2f47-b5d3-4&from=paste&height=583&id=u05ad4e93&originHeight=560&originWidth=1004&originalType=binary&ratio=0.9599999785423279&rotation=0&showTitle=false&size=24379&status=done&style=none&taskId=ueebc8ce3-af95-4f1a-adc5-b5d7e79b859&title=&width=1045.8333567095306)
初始默认登陆用户是`root`,密码通过以下命令获得：
```shell
sudo docker exec -it gitlab grep 'Password:' /etc/gitlab/initial_root_password
```
:::warning
这个密码存储24小时，过后将获取不到，请注意留存。
在10.168.1.99这台服务器上配置的github实例，root账户密码是：
`p5biOVz9UdA7wcAPfdgtLXVvuIEYeFfbI6kl9rQ5RDM=`
:::
# 备份与升级
## 备份
:::info
数据备份也是非常重要的部分，请务必重视。
:::
使用以下命令创建全量数据备份：
```shell
docker exec -t <container name> gitlab-backup create
```
![image.png](../asset/gitlabimagehost/1692842803444-c31ccd57-ca11-41f1-a681-223dc13ab1b3.png#averageHue=%23300a25&clientId=ua03159da-e778-4&from=paste&height=643&id=u2041c4ef&originHeight=617&originWidth=1006&originalType=binary&ratio=0.9599999785423279&rotation=0&showTitle=false&size=433117&status=done&style=none&taskId=uf6c15217-4102-43de-8806-5fbbd220b56&title=&width=1047.91669008943)
备份配置文件`gitlab.rb`和`gitlab-secrets.json`：
```shell
sudo cp $GITLAB_HOME/config/gitlab.rb $GITLAB_HOME/config/gitlab-secrets.json ~/
```
仅创建数据库备份：
（万一升级遇到问题，需要数据库备份来回滚极狐GitLab 升级。）
```shell
docker exec -t <container name> gitlab-backup create SKIP=artifacts,repositories,registry,uploads,builds,pages,lfs,packages,terraform_state
```
备份被写入` /var/opt/gitlab/backups`，挂载在宿主机`$GITLAB_HOME/config/data`。
## 升级
:::info
升级前请先备份
:::
停止gitlab容器：
```shell
sudo docker stop gitlab
```
移除现有容器：
```shell
sudo docker rm gitlab
```
拉取新镜像：
镜像tag视情况而定
```shell
sudo docker pull registry.gitlab.cn/omnibus/gitlab-jh:latest
```
使用之前的配置再次创建容器：
```shell
sudo docker run --detach \
  --hostname gitlab.virtualplane.com \
  --publish 443:443 --publish 80:80 --publish 23:22 \
  --name gitlab \
  --restart always \
  --volume $GITLAB_HOME/config:/etc/gitlab \
  --volume $GITLAB_HOME/logs:/var/log/gitlab \
  --volume $GITLAB_HOME/data:/var/opt/gitlab \
  --shm-size 256m \
  registry.gitlab.cn/omnibus/gitlab-jh:latest
```
查看容器状态直至`healthy`即可。
```shell
docker ps | grep gitlab
```
# 镜像仓库搭建
极狐GitLab 容器镜像库可以为每个项目都配置自己的空间来存储 Docker 镜像。
[About Registry](https://docs.docker.com/registry/introduction/)在此文档中了解更多 Docker Registry的详细信息。
## SSL证书
容器镜像库默认在 HTTPS 下工作。 可以使用 HTTP，但并不推荐。
由于镜像仓库运行在内网，可以通过内部CA签发证书或者使用自签名证书。
此处使用OpenSSL生成自签名证书：
```shell
# 由于权限问题，此处使用root权限执行
sudo su
cd ~
mkdir -p certs
openssl req \
  -newkey rsa:4096 -nodes -sha256 -keyout certs/domain.key \
  -addext "subjectAltName = DNS:registry.virtualplane.com" \
  -x509 -days 365 -out certs/domain.crt
```
:::info
此处的`subjectAltName = DNS:`需要替换为自己的仓库域名，比如我这里使用`registry.virtualplane.com`
:::
证书生成完了以后需要配置每个 Docker 守护进程信任该证书，不然在login的时候会报错。
Linux系统下需要将`domain.crt`复制到docker的证书目录下对于域名文件夹中。默认情况下：
```shell
cd /etc/docker/cert.d
# 注意域名
mkdir registry.virtualplane.com
cp domain.crt /etc/docker/cert.d/registry.virtualplane.com/ca.crt
```
不需要重启docker。
## Gitlab配置
将上一步生成的证书和密钥放入Gitlab的配置目录中，在宿主机上是`/srv/gitlab/config/ssl`（ssl目录没有就mkdir一个）,对应容器里的`/etc/gitlab/ssl`.
```shell
sudo su
cd ~/certs
cp domain.crt /srv/gitlab/config/ssl/registry.virtualplane.com.crt
cp domain.key /srv/gitlab/config/ssl/registry.virtualplane.com.key
```
确保证书权限：
```shell
chmod 600 /srv/gitlab/config/ssl/registry.gitlab.example.com.*
```
放置好 TLS 证书后，编辑虚拟机内的/etc/gitlab/gitlab.rb：
```shell
docker exec -it gitlab /bin/bash
vi /etc/gitlab/gitlab.rb
```
```shell
external_url 'https://gitlab.virtualplane.com'
registry_external_url 'https://registry.virtualplane.com'
gitlab_rails['registry_enabled'] = true
gitlab_rails['registry_host'] = "registry.virtualplane.com"
```
注意此处需写明`https`。
gitlab.rb中还有一些可能有用的设置，遇到问题时可以尝试设置如下信息尝试解决：
```shell
external_url 'https://gitlab.virtualplane.com'                                            
registry_external_url 'https://registry.virtualplane.com'
gitlab_rails['registry_enabled'] = true
gitlab_rails['registry_host'] = "registry.virtualplane.com"                                   
gitlab_rails['registry_port'] = "5000"
gitlab_rails['registry_api_url'] = "http://127.0.0.1:5000" 
registry_nginx['enable'] = true
registry['registry_http_addr'] = "127.0.0.1:5000"                                                                
registry['debug_addr'] = "localhost:5001" 
registry_nginx['listen_https'] = true                                                                    
registry_nginx['redirect_http_to_https'] = true  
```
保存配置，重新配置Gitlab实例使配置生效。
```shell
sudo docker restart gitlab
```
更多配置参考Gitlab文档[极狐GitLab 更多配置](https://docs.gitlab.cn/jh/administration/packages/container_registry.html#%E5%9C%A8%E7%AB%99%E7%82%B9%E8%8C%83%E5%9B%B4%E5%86%85%E7%A6%81%E7%94%A8%E5%AE%B9%E5%99%A8%E9%95%9C%E5%83%8F%E5%BA%93)
# 验证
以上配置正常的情况的下可以通过gitlab的用户名和密码登录：
```shell
docker login registry.virtualplane.com
# 输入用户名密码登录
```
登录成功：
![image.png](../asset/gitlabimagehost/1692845424573-d7eb9aa9-1aff-4b0f-8ab5-38e7ef9d60bf.png#averageHue=%23300b25&clientId=u5cf3c9fa-016b-4&from=paste&height=149&id=uf11148f0&originHeight=143&originWidth=793&originalType=binary&ratio=0.9599999785423279&rotation=0&showTitle=false&size=32949&status=done&style=none&taskId=u79196dc4-22ff-4059-8fb7-4a6e27f22c3&title=&width=826.0416851301371)
打个镜像上传仓库以验证可用性：
```shell
mkdir staticweb
cd staticweb
vim Dockerfile
```
写入以下Dockerfile：
```shell
# Version: 0.0.1
FROM ubuntu:latest
MAINTAINER Bourbon Tian "bourbon@1mcloud.com"
RUN apt-get update
RUN apt-get install -y nginx
RUN echo 'Hi, I am in your container' > /usr/share/nginx/html/index.html
EXPOSE 80
```
构建镜像
构建镜像时需要按照 域名/用户/项目名 的规则构建。如果不清楚可以访问自己项目-部署-容器镜像库的页面，会有构建提示，如图所示：
![屏幕截图 2023-11-23 111412.png](../asset/gitlabimagehost/1700709400776-67ac9e75-1509-4bc8-a415-81fe5f06ca62.png#averageHue=%23e6c499&clientId=u4c5661bb-d1fa-4&from=ui&id=u77d638ff&originHeight=900&originWidth=1905&originalType=binary&ratio=1&rotation=0&showTitle=false&size=103309&status=done&style=none&taskId=u837e5e99-0704-4d67-a006-9919ec400f6&title=)
需注意的是，docker login registry.virtualplane.com:5000 时可能会有问题，需要去掉":5000"，直接尝试login域名。下面build和pull时也不要带":5000"
```shell
# 根据项目提示构建docker image
docker build -t registry.virtualplane.com/internal/osmdev/testnginx .
```
![image.png](../asset/gitlabimagehost/1692845677146-dd940fb9-c172-4ce4-b62a-2a0956fbe095.png#averageHue=%232f0a25&clientId=u5cf3c9fa-016b-4&from=paste&height=419&id=ua72eda20&originHeight=402&originWidth=1194&originalType=binary&ratio=0.9599999785423279&rotation=0&showTitle=false&size=127714&status=done&style=none&taskId=ua07257f4-8678-4eb7-adc1-6f36bd08f92&title=&width=1243.7500277999795)
构建完成后查看一下：
```shell
$ docker images                                                                                
REPOSITORY                                            TAG       IMAGE ID       CREATED          SIZE
registry.virtualplane.com/internal/osmdev/testnginx   latest    20c701ebc91c   52 seconds ago   176MB
```
上传仓库：
```shell
docker push registry.virtualplane.com/internal/osmdev/testnginx
```
在仓库中查看：
![image.png](../asset/gitlabimagehost/1692845840488-06f697b7-7eba-4b37-bf47-58f7541e16ad.png#averageHue=%23fefefe&clientId=u5cf3c9fa-016b-4&from=paste&height=323&id=ub9e5da24&originHeight=310&originWidth=471&originalType=binary&ratio=0.9599999785423279&rotation=0&showTitle=false&size=16571&status=done&style=none&taskId=uefc410e2-b072-43de-a789-2502ab1363a&title=&width=490.62501096632354)
# 故障排查
安装过程中因为操作失误或者别的原因可能会有各种报错，通过检查输入命令和查看日志查明原因。
读取容器日志：
```shell
sudo docker logs gitlab
```
进入容器：
```shell
sudo docker exec -it gitlab /bin/bash
```
:::info
还有个可能会遇到的问题：Docker的[默认日志驱动是json-file](https://docs.docker.com/config/containers/logging/configure/#configure-the-default-logging-driver)，默认不执行日志轮换。因此长时间运行后会生成大量的日志数据消耗磁盘空间，需要注意清理。
:::
另外还可参考官方文档[极狐GitLab Docker 镜像 | 极狐GitLab](https://docs.gitlab.cn/jh/install/docker.html#%E6%9E%81%E7%8B%90gitlab-docker-%E9%95%9C%E5%83%8F)
