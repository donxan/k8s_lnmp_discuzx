###在kubernetes中搭建LNMP环境，并安装Discuz

本实验，需要已经搭建好kubernetes集群和harbor服务。

首先克隆本项目：git clone  https://github.com/donxan/k8s_lnmp_discuzx.git

####下载镜像
```
docker pull mysql:5.7
docker pull richarvey/nginx-php-fpm
```

####用dockerfile重建nginx-php-fpm镜像
```
cd k8s_discuz/dz_web_dockerfile/
docker build -t nginx-php .

```

####将镜像push到harbor
```
##登录harbor，并push新的镜像
docker login harbor.abcgogo.com  //输入正确的用户名和密码,如果之前已创建secret，就可以免输入密码
docker tag nginx-php  harbor.abcgogo.com/lnmp/nginx-php
docker push harbor.abcgogo.com/lnmp/nginx-php
docker tag mysql:5.7 harbor.abcgogo.com/lnmp/mysql:5.7
docker push harbor.abcgogo.com/lnmp/mysql:5.7
```

####搭建NFS
可以参考以下：

假设kubernetes集群网段为192.168.2.0/24，本机IP为192.168.2.10
* 安装包
```
yum install nfs-utils
```
* 编辑配置文件
```
vim /etc/exportfs  //内容如下
/data/k8s/ 192.168.2.0/24(sync,rw,no_root_squash)
```
* 启动服务
```
systemctl start nfs
systemctl enable nfs
```
* 创建目录
```
mkdir -p  /data/k8s/discuz/{db,web}
```

####搭建MySQL服务
* 创建secret (设定mysql的root密码)
```
kubectl create secret generic mysql-pass --from-literal=password=DzPasswd1
```
* 创建pv
```
cd ../../k8s_discuz/mysql
kubectl apply -f mysql-pv.yaml
```
* 创建pvc
```
kubectl apply -f mysql-pvc.yaml
```
* 创建deployment
```
kubectl apply -f mysql-dp.yaml 
```
* 创建service
```
kubectl apply -f mysql-svc.yaml
```

####搭建Nginx+php-fpm服务
注意搭建步骤，在部署mysql时，不能deploy，svc一起执行，需要一步一步来操作。
* 搭建pv
```
cd ../../k8s_discuz/nginx_php
kubectl apply -f web-pv.yaml
```
* 创建pvc
```
kubectl apply -f web-pvc.yaml
```
* 创建deployment
```
kubectl apply -f web-dp.yaml 
```
* 创建service
```
kubectl apply -f web-svc.yaml
```
####安装Discuz
* 下载dz代码 (到NFS服务器上)
```
cd /tmp/
git clone https://gitee.com/ComsenzDiscuz/DiscuzX.git
cd /data/k8s/discuz/web/
mv /tmp/DiscuzX/upload/* .
chown -R 100 data uc_server/data/ uc_client/data/ config/
```
* 设置MySQL普通用户
```
kubectl get svc dz-mysql //查看service的cluster-ip，我的是10.68.122.120
mysql -uroot -h10.68.122.120 -pDzPasswd1  //这里的密码是在上面步骤中设置的那个密码
> create database dz;
> grant all on dz.* to 'dz'@'%' identified by 'dz-passwd-123';
```
#### 部署traefik ingress
```
cd ../../k8s_discuz/nginx_php
kubectl apply -f web-ingress.yaml
```
如果没有部署ingress，可以使用安装nginx,配置nginx反向代理。
参考如下。

* 设置Nginx代理
```
注意：目前nginx服务是运行在kubernetes集群里，node节点以及master节点上是可以通过cluster-ip访问到，但是外部的客户端就不能访问了。
      所以，可以在任意一台node或者master上建一个nginx反向代理即可访问到集群内的nginx。
kubectl get svc dz-web //查看cluster-ip，我的ip是10.68.190.99
nginx代理配置文件内容如下：
server {
            listen 80;
            server_name dz.abcgogo.com;

            location / {
                proxy_pass      http://10.68.190.99:80;
                proxy_set_header Host   $host;
                proxy_set_header X-Real-IP      $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
}
```

* 安装Discuz
```
做完Nginx代理，就可以通过node的IP来访问discuz了。
```