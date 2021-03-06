# k8s flannel 网络问题 #

k8s部署完成后，使用deployment分别在两个node上部署一个容器，发现node1节点docker0可以ping通node2节点的docker0，但是容器无法连通。如下：

![](https://github.com/Hanzhiwei210521/loading/blob/master/image/image009.png)

原因为数据包到目的主机的flannel0后没有发送到docker0上，主要是由于forward chain中Action为DOCKE的规则阻断了数据包进入docker0。

有一个问题值得思考，我系统是mini安装，默认没有安装iptables和firewalld,docker是怎么怎么实现的这些规则，百思不得姐！

先安装iptables

    [root@k8s-node2 ~]# yum install iptables-services
启动iptables加入开机自启，并重启docker

    [root@k8s-node2 ~]# systemctl start iptables
    [root@k8s-node2 ~]# systemctl enable iptables
    [root@k8s-node2 ~]# systemctl restart docker

查看iptables规则：

    [root@k8s-node2 ~]# iptables -nL -v
    Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
    pkts bytes target     prot opt in     out     source               destination         
    78244 9264K KUBE-SERVICES  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service portals */
    78531 9295K KUBE-FIREWALL  all  --  *      *       0.0.0.0/0            0.0.0.0/0           
    79375 9392K ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            state  RELATED,ESTABLISHED
        0     0 ACCEPT     icmp --  *      *       0.0.0.0/0            0.0.0.0/0           
        1    60 ACCEPT     all  --  lo     *       0.0.0.0/0            0.0.0.0/0           
        0     0 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            state NEW tcp dpt:22
        0     0 REJECT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            reject-with icmp-host-prohibited

    Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
     pkts bytes target     prot opt in     out     source               destination         
        0     0 DOCKER-USER  all  --  *      *       0.0.0.0/0            0.0.0.0/0           
        0     0 DOCKER-ISOLATION  all  --  *      *       0.0.0.0/0            0.0.0.0/0           
        0     0 ACCEPT     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
        0     0 DOCKER     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0           
        0     0 ACCEPT     all  --  docker0 !docker0  0.0.0.0/0            0.0.0.0/0           
        0     0 ACCEPT     all  --  docker0 docker0  0.0.0.0/0            0.0.0.0/0           
       24  1728 KUBE-FORWARD  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes forward rules */
       24  1728 REJECT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            reject-with icmp-host-prohibited

    Chain OUTPUT (policy ACCEPT 3400 packets, 323K bytes)
     pkts bytes target     prot opt in     out     source               destination         
    78360 6633K KUBE-SERVICES  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service portals */
    78643 6657K KUBE-FIREWALL  all  --  *      *       0.0.0.0/0            0.0.0.0/0           

    Chain DOCKER (1 references)
     pkts bytes target     prot opt in     out     source               destination         

    Chain DOCKER-ISOLATION (1 references)
     pkts bytes target     prot opt in     out     source               destination         
        0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0           

    Chain DOCKER-USER (1 references)
     pkts bytes target     prot opt in     out     source               destination         
        0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0           

    Chain KUBE-FIREWALL (2 references)
     pkts bytes target     prot opt in     out     source               destination         
        0     0 DROP       all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes firewall for dropping marked packets */ mark match 0x8000/0x8000

    Chain KUBE-FORWARD (1 references)
     pkts bytes target     prot opt in     out     source               destination         

    Chain KUBE-SERVICES (2 references)
     pkts bytes target     prot opt in     out     source               destination
可以看到上述规则中FORWARD中的规则：

     0     0 DOCKER     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0   
该规则将数据包交给DOCKER chain处理，而docker chain没有放行，就是说，缺省情况下docker service是不容许docker0外部数据经过docker0进入docker内部，所以在docker chain中增加规则：

    iptables -I DOCKER -s 0.0.0.0/0 -d 0.0.0.0/0 ! -i docker0 -o docker0 -j ACCEPT
但是重启后，由于docker服务自身会刷新该规则，会导致增加的规则丢失，所以直接增加到FORWARD chain中：

    iptables -I FORWARD -s 0.0.0.0/0 -d 0.0.0.0/0 ! -i docker0 -o docker0 -j ACCEPT
检查输出：

    Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
     pkts bytes target     prot opt in     out     source               destination         
        0     0 ACCEPT     all  --  !docker0 docker0  0.0.0.0/0            0.0.0.0/0           
       17  1292 DOCKER-USER  all  --  *      *       0.0.0.0/0            0.0.0.0/0           
       17  1292 DOCKER-ISOLATION  all  --  *      *       0.0.0.0/0            0.0.0.0/0           
        0     0 ACCEPT     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
        0     0 DOCKER     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0           
       17  1292 ACCEPT     all  --  docker0 !docker0  0.0.0.0/0            0.0.0.0/0           
        0     0 ACCEPT     all  --  docker0 docker0  0.0.0.0/0            0.0.0.0/0           
       24  1728 KUBE-FORWARD  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes forward rules */
       24  1728 REJECT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            reject-with icmp-host-prohibited


强制在机器启动时在forward链中增加一条转发规则：

1、写一个脚本
    
    [root@k8s-node2 ~]# cat /etc/sysconfig/add-forward-iptable-rule.sh 
    #!/bin/bash
    sleep 10
    iptables -I FORWARD  -s 0.0.0.0/0 -d 0.0.0.0/0  -j ACCEPT
2、在systemd中增加一个service并开机启动

    [root@k8s-node2 ~]# cat /usr/lib/systemd/system/add-iptable-rule.service 
    [Unit]
    Description=enable forward all for forward chain in filter table
    After=docker.service
    [Service]
    ExecStart=/bin/bash /etc/sysconfig/add-forward-iptable-rule.sh
    [Install]
    WantedBy=multi-user.target
3、加入开机自启

    [root@k8s-node2 sysconfig]# systemctl enable add-iptable-rule.service 
    Created symlink from /etc/systemd/system/multi-user.target.wants/add-iptable-rule.serv
到这里，你以为万事大吉了，oh,no,no,no，到这里，问题并没有解决，而是变得更复杂了，现在node1节点上的容器和node2节点上的容器，单向ping不同，只有容器1ping容器2,然后容器2同时ping容器1才通，只要通了这一次，在一定时间内（大概一两分钟），单向ping是可以的，过了这个时间，单向ping依然不通，双方同时ping依然可以，而且，双方的docker0网桥一直处于无法ping通状态。问题解决，原来是iptables把flannel的端口封掉了，添加iptables允许flannel端口即可。

     [root@k8s-node2 ~]# cat /etc/sysconfig/add-forward-iptable-rule.sh 
     #!/bin/bash
     sleep 10
     iptables -I INPUT -p tcp --dport 2379 -j ACCEPT
     iptables -I INPUT -p tcp --dport 2380 -j ACCEPT
     iptables -I INPUT -p udp --dport 8285 -j ACCEPT
     iptables -I FORWARD -s 0.0.0.0/0 -d 0.0.0.0/0 ! -i docker0 -o docker0 -j ACCEPT
端口2379和2380是etcd端口，8285是flannel UDP协议端口，VxLan使用8472端口，这里使用的是UDP协议。

文章参考： <https://www.myf5.net/post/2304.htm>