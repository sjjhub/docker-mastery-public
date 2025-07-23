# docker交互原理图
![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/image.png\)

# run运行流程图

![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/image-1.png\)
![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/image-2.png\)

# Docker为什么比VM运行快？
1、docker有着比虚拟机更少的抽象层  
2、docker利用的是宿主机的内核，M需要的是Guest OS  
//原理图  
![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/image-3.png\)

#  常用命令
```
cat /etc/os-release  
docker --version  
docker  info  
docker images  (-a,-q)  
```
# 镜像命令
```
docker search  
docker  pull  
docker rmi  -f uid  
```
#  容器命令
说明：我们有了镜像才可以创建容器，下载一个centos镜像来测试学习  
docker pull  centos  
新建容器并启动镜像  
```  
docker run  【可选参数】 image  
--name   容器名字   
-d    后台方式运行  
-it   使用交互方式运行，进入容器看内容  
-P    指定容器的端口， -P  8080:8080  
     -P  ip:主机端口：容器端口   
     -P  主机端口：容器端口   
     -P 容器端口  
-p    随机指定端口  
```  
测试、启动进入容器  
![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/image-4.png\)  
从容器中退回主机  
![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/image-5.png\)  
列出所有运行的容器   
```  
docker  ps  命令  
空白    列出当前正在运行的容器  
-a      列出当前运行的容器+历史运行过的容器  
-n      列出最近创建的容器   
-q      只显示容器的编号    
```  
// 运行结果图  
![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/image-6.png\)  
退出容器  
``` 
exit  容器停止并退出
ctrl+p+Q    容器不停止退出  
```
删除容器
```
docker rm  容器id   删除指定容器，不能删除正在运行的容器
docker rm -f $(docker ps -aq)  删除所有容器
docker ps -a -q |xargs docker rm  删除所有容器
```
启动停止容器的操作
``` 
docker start 容器id
docker restart 容器id 
docker stop 容器id 
docker kill 容器id
```
常用其他命令  
后台启动容器  
常见的坑，docker容器使用后台运行，就必须有一个前台进程，docker发现没有应用，会自动停止  
nginx,容器启动后，发现自己没有提供服务，就会立刻停止，就是没有程序了  
```
docker run -d  镜像名
```   
问题：docker ps发现centos停止了   
![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/image-7.png\)  
查看日志  
```
docker logs
docker logs  -f -t --tail 10
docker logs -tf --tail 10 4d1f9a5f5dcf   显示指定行数的日志 
            -tf   显示日志
         --tail  number  要显示日志的条数
```  
shell脚本：  
"while true;do echo sjj ;sleep  1;done"    #无限循环打印sjj  
查看容器中的进程信息
```
docker top  容器id 
```
![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/image-8.png\)  
查看镜像的源数据
```
docker inspect   容器id 
[admin@localhost ~]$ docker inspect 4d1f9a5f5dcf
[
    {
        "Id": "4d1f9a5f5dcf17455159454969bb909f95722825b130f8e5c21e66ee7f286124",
        "Created": "2025-07-06T08:16:42.432913952Z",
        "Path": "/bin/bash",
        "Args": [
            "-c",
            "while true;do echo sjj ;sleep  1;done"
        ],
        "State": {
            "Status": "running",
            "Running": true,
            "Paused": false,
            "Restarting": false,
            "OOMKilled": false,
            "Dead": false,
            "Pid": 12621,
            "ExitCode": 0,
            "Error": "",
            "StartedAt": "2025-07-06T08:16:42.701991851Z",
            "FinishedAt": "0001-01-01T00:00:00Z"
        },
        "Image": "sha256:5d0da3dc976460b72c77d94c8a1ad043720b0416bfc16c52c45d4847e53fadb6",
        "ResolvConfPath": "/var/lib/docker/containers/4d1f9a5f5dcf17455159454969bb909f95722825b130f8e5c21e66ee7f286124/resolv.conf",
        "HostnamePath": "/var/lib/docker/containers/4d1f9a5f5dcf17455159454969bb909f95722825b130f8e5c21e66ee7f286124/hostname",
        "HostsPath": "/var/lib/docker/containers/4d1f9a5f5dcf17455159454969bb909f95722825b130f8e5c21e66ee7f286124/hosts",
        "LogPath": "/var/lib/docker/containers/4d1f9a5f5dcf17455159454969bb909f95722825b130f8e5c21e66ee7f286124/4d1f9a5f5dcf17455159454969bb909f95722825b130f8e5c21e66ee7f286124-json.log",
        "Name": "/great_goldwasser",
        "RestartCount": 0,
        "Driver": "overlay2",
        "Platform": "linux",
        "MountLabel": "",
        "ProcessLabel": "",
        "AppArmorProfile": "",
        "ExecIDs": null,
        "HostConfig": {
            "Binds": null,
            "ContainerIDFile": "",
            "LogConfig": {
                "Type": "json-file",
                "Config": {}
            },
            "NetworkMode": "bridge",
            "PortBindings": {},
            "RestartPolicy": {
                "Name": "no",
                "MaximumRetryCount": 0
            },
            "AutoRemove": false,
            "VolumeDriver": "",
            "VolumesFrom": null,
            "ConsoleSize": [
                24,
                86
            ],
            "CapAdd": null,
            "CapDrop": null,
            "CgroupnsMode": "host",
            "Dns": [],
            "DnsOptions": [],
            "DnsSearch": [],
            "ExtraHosts": null,
            "GroupAdd": null,
            "IpcMode": "private",
            "Cgroup": "",
            "Links": null,
            "OomScoreAdj": 0,
            "PidMode": "",
            "Privileged": false,
            "PublishAllPorts": false,
            "ReadonlyRootfs": false,
            "SecurityOpt": null,
            "UTSMode": "",
            "UsernsMode": "",
            "ShmSize": 67108864,
            "Runtime": "runc",
            "Isolation": "",
            "CpuShares": 0,
            "Memory": 0,
            "NanoCpus": 0,
            "CgroupParent": "",
            "BlkioWeight": 0,
            "BlkioWeightDevice": [],
            "BlkioDeviceReadBps": [],
            "BlkioDeviceWriteBps": [],
            "BlkioDeviceReadIOps": [],
            "BlkioDeviceWriteIOps": [],
            "CpuPeriod": 0,
            "CpuQuota": 0,
            "CpuRealtimePeriod": 0,
            "CpuRealtimeRuntime": 0,
            "CpusetCpus": "",
            "CpusetMems": "",
            "Devices": [],
            "DeviceCgroupRules": null,
            "DeviceRequests": null,
            "MemoryReservation": 0,
            "MemorySwap": 0,
            "MemorySwappiness": null,
            "OomKillDisable": false,
            "PidsLimit": null,
            "Ulimits": [],
            "CpuCount": 0,
            "CpuPercent": 0,
            "IOMaximumIOps": 0,
            "IOMaximumBandwidth": 0,
            "MaskedPaths": [
                "/proc/asound",
                "/proc/acpi",
                "/proc/kcore",
                "/proc/keys",
                "/proc/latency_stats",
                "/proc/timer_list",
                "/proc/timer_stats",
                "/proc/sched_debug",
                "/proc/scsi",
                "/sys/firmware",
                "/sys/devices/virtual/powercap"
            ],
            "ReadonlyPaths": [
                "/proc/bus",
                "/proc/fs",
                "/proc/irq",
                "/proc/sys",
                "/proc/sysrq-trigger"
            ]
        },
        "GraphDriver": {
            "Data": {
                "LowerDir": "/var/lib/docker/overlay2/f6ea620a8e9edaf611255682ea67f270bc430a3e94033a10acc75a3f888797d7-init/diff:/var/lib/docker/overlay2/21a7f9c632c5e112335ae398a6862a913fde3d6632f92eba4d1d36598e27accc/diff",
                "MergedDir": "/var/lib/docker/overlay2/f6ea620a8e9edaf611255682ea67f270bc430a3e94033a10acc75a3f888797d7/merged",
                "UpperDir": "/var/lib/docker/overlay2/f6ea620a8e9edaf611255682ea67f270bc430a3e94033a10acc75a3f888797d7/diff",
                "WorkDir": "/var/lib/docker/overlay2/f6ea620a8e9edaf611255682ea67f270bc430a3e94033a10acc75a3f888797d7/work"
            },
            "Name": "overlay2"
        },
        "Mounts": [],
        "Config": {
            "Hostname": "4d1f9a5f5dcf",
            "Domainname": "",
            "User": "",
            "AttachStdin": false,
            "AttachStdout": false,
            "AttachStderr": false,
            "Tty": false,
            "OpenStdin": false,
            "StdinOnce": false,
            "Env": [
                "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
            ],
            "Cmd": [
                "/bin/bash",
                "-c",
                "while true;do echo sjj ;sleep  1;done"
            ],
            "Image": "centos",
            "Volumes": null,
            "WorkingDir": "",
            "Entrypoint": null,
            "OnBuild": null,
            "Labels": {
                "org.label-schema.build-date": "20210915",
                "org.label-schema.license": "GPLv2",
                "org.label-schema.name": "CentOS Base Image",
                "org.label-schema.schema-version": "1.0",
                "org.label-schema.vendor": "CentOS"
            }
        },
        "NetworkSettings": {
            "Bridge": "",
            "SandboxID": "eac381394f703e9da2ae73c2137d4d3668d5c5cae5fc1213e37f2a44b459e2c3",
            "SandboxKey": "/var/run/docker/netns/eac381394f70",
            "Ports": {},
            "HairpinMode": false,
            "LinkLocalIPv6Address": "",
            "LinkLocalIPv6PrefixLen": 0,
            "SecondaryIPAddresses": null,
            "SecondaryIPv6Addresses": null,
            "EndpointID": "f0dd3670d4deaa0745cda0e9805a723e0228c7d0bc175ec93faf0d5da1d32959",
            "Gateway": "172.17.0.1",
            "GlobalIPv6Address": "",
            "GlobalIPv6PrefixLen": 0,
            "IPAddress": "172.17.0.2",
            "IPPrefixLen": 16,
            "IPv6Gateway": "",
            "MacAddress": "02:42:ac:11:00:02",
            "Networks": {
                "bridge": {
                    "IPAMConfig": null,
                    "Links": null,
                    "Aliases": null,
                    "MacAddress": "02:42:ac:11:00:02",
                    "NetworkID": "ca4ad4d75eb30d90a748360bd8cecc7eff52be0780a73fcb13e5ac94be7ff6be",
                    "EndpointID": "f0dd3670d4deaa0745cda0e9805a723e0228c7d0bc175ec93faf0d5da1d32959",
                    "Gateway": "172.17.0.1",
                    "IPAddress": "172.17.0.2",
                    "IPPrefixLen": 16,
                    "IPv6Gateway": "",
                    "GlobalIPv6Address": "",
                    "GlobalIPv6PrefixLen": 0,
                    "DriverOpts": null,
                    "DNSNames": null
                }
            }
        }
    }
]
```
进入当前正在运行的容器  
通常容器都是使用后台方式运行的，需要进入容器，修改一些配置  
```
docker exec -it   容器id bashshell
docker   attach 容器id    #正在执行当前的代码…
```
区别：  
docker exec 进入容器后开启一个新的开端，可以在里面操作(常用）  
docker attach   进入容器正在执行的终端，不会启动新的进程  
从容器内拷贝文件到主机  
```
docker cp 容器id:容器内路径  目的主机路径
```
拷贝是一个手动过程，未来我们使用-v卷的技术，可以实现自动同步  
![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/image-9.png\)  
```
搜索镜像:docker search nginx
下载镜像：docker pull nginx
```
运行测试  
```
docker   run 
-d 后台运行
--name  给容器命名
-p  宿主机端口，容器内部端口
```
![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/image-10.png\)
![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/image-11.png\)  
 端口暴露概念  
 ![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/image-12.png\)  
 思考问题：我们每次改动nginx配置文件，都需要进入容器内部？十分的麻烦，我要是可以在容器外部提供一个映射路径，达到在容器修改文件名，容器内部就可以自动修改？  
 docker安装一个tomcat  
 docker run -it  --rm  tomcat:9.0   容器用完即删  
 我们之前启动的都是后台，停止了容器之后，容器还是可以查到   
 下载再启动  
 docker pull tomcat   
 启动运行  
 ```
 docker  run -d  -p  3355:8080 --name  tomcat01  tomcat
 ```
 进入容器  
 ![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/image-13.png\)  
 发现问题linux命令少了，没有webapps,阿里云镜像的原因，默认是最小的镜像，所有不必要的都剔除掉，保证最小可运行环境  
 ![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/image-14.png\)   
 思考问题：我们以后要部署项目，如果每次都要进容器是不是十分麻烦？我要是可以在容器外部提供一个映射路径，webapps,我们再外部放置项目，就自动同步到内部就好了！  
 docker容器tomcat+网站  
 部署es+kibana  
 es:暴露端口很多  
 es:十分耗内存   
 es:的数据一般需要放置到安全目录，挂载   
 官方命令  
 ```
 docker run -d   --name  elasticsearch --net somenetwork -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" elasticsearch:tag
 ```
 启动 elasticsearch   
 ```
 docker run -d --name elasticsearch -p 9200:9200 -p 9300:9300 -e  "discovery.type=single-node"  elasticsearch:7.6.2
 ```
 启动了linux就卡住了   
 ```
  docker  stats     #查看cpu状态
  ```
  ![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/image-15.png\)   
  测试es是否成功  
  ```
  curl localhost:9200
  ```
  ![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/image-16.png\)  
  查看下载es后的docker  stats  cpu内存   
 ![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/image-18.png\) 
  赶紧关闭，增加内存的限制，修改配置文件  -e  环境配置修改  
  ```
  docker  run -d --name  elasticsearch -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" -e ES_JAVA_OPTS="-Xms64m -Xmx512m" elasticsearch:7.6.2
  ```
  作业:使用kibanna连接es？思考网络如何才能连接过去   
  ![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/image-19.png\)  
  ### Docker可视化  
  portainer(先用这个）
  ```
  docker run -d -p 8088:9000 \
  --restart=always -v /var/run/docker.sock:/var/run/docker.sock --privileged=true portainer/portainer
  ```
  Rancher (CI/CD再用）  
  什么是portainer?  
  docker图形化界面管理工具!提供一个后台面板供我们操作  
```
  docker run -d \
    --name=portainer \
    -p 8088:9000 \
    --restart=always \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v portainer_data:/data \
    portainer/portainer-ce:latest
    参数说明：
    • -d：后台运行
    • --name=portainer：容器名称
    • -p 8088:9000：将本地 8088 端口映射到 Portainer 的 9000 端口
    • --restart=always：Docker 重启时自动启动 Portainer
    • -v /var/run/docker.sock:/var/run/docker.sock：挂载 Docker 套接字（必须，用于管理 Docker）
    • -v portainer_data:/data：持久化存储 Portainer 数据（避免重启后丢失配置）
    portainer/portainer-ce:latest：使用 Portainer Community Edition（官方推荐）
```   
测试结果：
    ![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/image-20.png\) 
     



     若要升级portainer:
   
     # 停止并删除旧容器
    docker rm -f portainer
    
    # 删除旧数据卷（谨慎操作！会丢失所有配置）
    docker volume rm portainer_data
    
    # 重新运行 Portainer（使用新数据卷）
    docker run -d \
      --name=portainer \
      -p 8088:9000 \
      --restart=always \
      -v /var/run/docker.sock:/var/run/docker.sock \
      -v portainer_data:/data \
      portainer/portainer-ce:2.27.0

# docker镜像加载原理
unionfs(联合文件系统）  
我们下载的时候看到一层一层的就是这个
对于一个精简的os，rootfs可以很小，只需要包含基本的命令，工具和程序库就可以了，因为底层直接用host的kernel，自己只要提供rootfs就可以了。由此可见对于不同的linux发行版，bootfs基本是一致的，rootfs会有差别，因此不同的发行版本可以公用bootf  
虚拟机是分钟级别，容器是秒钟级别  
![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/image-21.png\)  
docker镜像都是只读的，当容器启动时，一个新的可写层被加载到镜像的顶部，这一层就是我们通常说的容器层，容器之下的都叫镜像层  
###  如何提交一个自己的镜像
commit镜像  
docker commit  提交容器成为一个新的副本  
命令和git原理类似  
```
docker  commit  -m="提交的描述信息"  -a="作者"  容器id  目标镜像名:[TAG]
```
###  实战测试
启动一个默认的tomcat  
发现这个默认的tomcat是没有webapps应用的，镜像的原因，官方的镜像默认webapps下面是没有文件的  
我现在拷贝进去了基本的文件  
把自己修改过后的镜像提交发布为一个新的镜像  
```
docker commit -a="sjj" -m="add webapps app"  c923d24bd399  tomcat01:1.0
```
![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/image-22.png\)  
如果想要保存当前容器的状态，可以通过commit来提交，获得一个镜像  

# 容器数据卷
什么是容器数据卷?  
将应用和环境打包成一个镜像  
数据？如果数据都在容器中，那么我们删除容器，数据就会丢失，需求：数据可以持久化  
mysql,容器删了，删库跑路了，需求：mysql数据可以存储在本地；容器之间可以有一个数据共享的技术，docker 容器中国钟产生的数据，同步到本地
这就是卷技术！目录的挂载  
容器的持久化和同步操作！容器间也是可以数据共享的  
使用数据卷  
直接使用命令来挂载   
```
docker run -it -v 主机目录，容器目录
```
测试  
```
docker  run -it  -v /home/ceshi:/home centos /bin/bash
```
启动起来的时候可以通过docker inspect 容器id查看  
![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/image-23.png\)  
容器内：  
![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/image-24.png\)  
容器外：
![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/image-25.png\)  
测试  
1、停止容器  
2、宿主机上修改文件  
3、启动容器   
4、容器内的数据依旧是同步的   
好处：以后修改只需要载本地修改即可，容器内会自动同步  
### 实战安装Mysql  
思考mysql持久化问题   
![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/image-26.png\)  
运行容器，需要做数据挂载，安装启动mysql，需要配置密码的，这是要注意的  
```
docker  run -d -p 3310:3306 -v /home/mysql/conf:/etc/mysql/conf.d -v /home/mysql/data:/var/lib/mysql   -e  MYSQL_ROOT_PASSWORD=mysql@2025  --name mysql01  mysql:5.7

参数说明：
-d  后台运行
-p  端口映射
-v  卷挂载
-e  环境配置
--name  容器名字
```
启动成功后，在本地使用navicat测试成功  
![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/image-27.png\)  
在本地测试创建一个数据库，查看映射路径是否Ok  
![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/image-28.png\)  
![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/image-29.png\)  
假设将容器删除  
发现挂载到本地的数据卷依旧没有丢失，这就实现了容器数据持久化功能  
![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/image-30.png\)  
![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/image-31.png\)
### 匿名挂卷  
```
docker run -d -P --name nginx01 -v /etc/nginx nginx
```
查看所有的volume的情况
```
docker volume   ls
```
![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/1752426265395_image.png\)
这里发现，这种就是匿名挂载，我们在-v只写了容器内的路径，没有写容器外的路径  
###  具名挂载
```
docker run -d -P --name nginx02 -v juming-nginx:/etc/nginx nginx
```
![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/1752426229278_image.png\)
通过-v  卷名：容器内路径  
查看一下这个卷：docker volume inspect juming-nginx  
![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/1752426130352_image.png\)  
所有的docker容器内的卷，没有指定目录的情况下都是在/var/lib/docker/volumes/xxxx/data  
![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/1752427048868_image.png\)
我们通过具名挂载可以方便的找到我们的一个卷，大多数情况下使用的具名挂载  
如何确定是具名挂载还是匿名挂载，还是指定路径挂载  
-v  容器内路径                     匿名挂载  
-v  卷名：容器内路径           具名挂载   
-v  /宿主机路径：/容器内路径     指定路径挂载  
拓展：  
通过  -v容器内路径:ro,rw，改变读写权限   
ro  readonly   只读，只能通过宿主机操作，容器内部无法操作  
rw readwrite  可读可写   
一旦这个设置了容器权限，容器对我们挂载出来的内容就有限定了   
```
docker run -d -P  --name nginx02 -v juming-nginx:/etc/nginx:ro nginx
docker run -d -P  --name nginx02 -v juming-nginx:/etc/nginx:rw nginx
```
# 初始dockerfile
dockerfile就是用来构建docker镜像的构建文件  
通过这个脚本可以 生成镜像，镜像是一层一层的，脚本一个个的命令，每个命令都是一层  
创建一个dockerfile文件，名字可以随机，建议dockerfile  
文件中的内容，指令大写，参数  
![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/1752427316339_image.png\)  
这里的每个命令，就是镜像的一层  
docker build -f <Dockerfile路径> -t <镜像名:标签> <构建上下文路径>  
```
docker build -f /home/docker-test-volume/dockerfile1 -t sjj/centos:1.0 .
```
![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/1752427373980_image.png\)   
启动自己写的镜像  
docker run -it   镜像id  
![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/1752427410825_image.png\)   
这个卷和外部一定有一个同步的目录  
docker inspect 容器id   
![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/1752427430660_image.png\)   
测试是否同步出去  
![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/1752427447571_image.png\)  
这种方式未来的使用的特别多，因为我们通常会构建自己的镜像；假设构建镜像的时候没有挂载卷，要手动镜像挂载  -v  卷名:容器内路径   

# 数据卷容器
两个mysql同步数据  
![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/1752427499927_image.png\)  
启动3个容器，通过我们刚才自己的写镜像启动  
![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/1752427527288_image.png\)  
![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/1752427537697_image.png\)  
测试，删除容器docker01，查看docker01和docker02依旧可以访问这两个文件  
![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/1752427556496_image.png\)  
多个mysql实现数据共享  
```
docker  run -d -p 3310:3306 -v /home/mysql/conf:/etc/mysql/conf.d -v /home/mysql/data:/var/lib/mysql   -e  MYSQL_ROOT_PASSWORD=mysql@2025  --name mysql01  mysql:5.7    

docker run -d -p 3310:3306 -e MYSQL_ROOT_PASSWORD=mysql@2025 --name mysql02 --volumes-from mysql01 mysql:5.7
```
这时候可以实现两个容器数据同步  
结论：容器之间配置信息的传递，数据卷容器的生命周期一致持续到没有人使用为止，但是数据持久化到本地，本地数据不会删除的！  
#  Dockerfile
### Dockerfile介绍
dockerfile是用来构建镜像的文件，命令参数脚本  
构建步骤：  
1、编写一个dockerfile文件  
2、dockerfile  build构建成为一个镜像  
3、docker run运行镜像  
4、docker push 发布镜像（dockerhub,阿里云镜像）  
很多官方镜像都是基础包，很多功能没有，我们通常会自己搭建自己的镜像  
###  Dockerfile构建过程
基础知识：  
1、每个保留关键字（指令）都是必须是大写字母  
2、从上到下顺序执行  
3、#表示注释  
4、每一个指令都会创建提交一个新的镜像层，并提交  
![]\(https://cdn.jsdelivr.net/gh/sjjhub/docker-mastery-public@master/images/1752427796940_image.png\)   
docker file是面向开发的，我们以后要发布项目，做镜像，就需要编写  dockerfile文件，这个文件十分简单  
dockerfile镜像逐渐成为企业交付的标准，必须要掌握  
步骤：开发，部署，运维   
dockerfile：构建文件，定义了一切的步骤，源代码   
dockerimages:通过dockerfile构建生成的镜像，最终发布和运行的产品，原来是jar，war  
docker容器：容器就是镜像运行起来提供的服务器  