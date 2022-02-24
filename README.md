# AWS中国区域高可用VPN方案

## 1. 方案部署
### 1.1 从新建VPC快速启动

在CloudFormation中导入以下模板，启动本地模拟环境

```
openswan-on-prem-test-new-vpc.yaml
```

在CloudFormation中导入以下模板，在新的VPC中搭建高可用VPN

```
openswan-main-china-new-vpc.yaml
```

### 1.2 从已有VPC中启动

>前提条件
><br>1. 已有VPC，绑定Internet Gateway
><br>2. 在两个可用区分别创建两个公有子网，并配置指向Internet Gateway的默认路由
><br>3. 公有子网配置为“启用自动分配公有 IPv4 地址”

在CloudFormation中导入以下模板，启动本地模拟环境

```
openswan-on-prem-test-existing-vpc.yaml
```
本地模拟环境的CloudFormation参数参考下图

![Alt text](https://github.com/toreydai/ha-vpn-aws-china/blob/main/images/cloudformation_op_test.png)

在CloudFormation中导入以下模板，在已有VPC中搭建高可用VPN

```
openswan-main-china-existing-vpc.yaml

```
高可用VPN环境的CloudFormation参数参考下图

![Alt text](https://github.com/toreydai/ha-vpn-aws-china/blob/main/images/cloudformaton_ha_vpn.png)

## 2.VPN配置

### 2.1 本地模拟环境

检查并配置/etc/ipsec.d/ipsec.conf，需要修改right的值为对端VPN实例绑定的EIP

```
config setup
        dumpdir=/var/run/pluto/
        nat_traversal=yes
        virtual_private=%v4:10.0.0.0/8,%v4:192.168.0.0/16,%v4:172.16.0.0/12,%v4:25.0.0.0/8,%v6:fd00::/8,%v6:fe80::/10
        oe=off
        protostack=netkey
        nhelpers=0
        interfaces=%defaultroute
conn openswan
        authby=secret
        type=tunnel
        auto=start

        left=172.16.3.8
        leftid=71.132.9.58
        leftsubnet=172.16.0.0/16
        leftnexthop=%defaultroute

        right=161.189.197.109
        rightsubnet=10.100.0.0/16
        rightnexthop=%defaultroute

        ike=aes256-sha1;modp1024
        ikelifetime=86400s
        phase2=esp
        phase2alg=aes256-sha1;modp1024
        salifetime=3600s
        pfs=yes
```
检查并配置/etc/ipsec.d/ipsec.secrets，修改第二个IP为对端VPN实例绑定的EIP

```
71.132.9.58  161.189.197.109 : PSK "psk123"
```
重新启动IPsec服务

```
sudo systemctl restart ipsec
```
### 2.2 高可用VPN环境

检查并配置/etc/ipsec.d/ipsec.conf, 需要修改right的值为本地模拟环境VPN实例的公有IP

```
config setup
        dumpdir=/var/run/pluto/
        nat_traversal=yes
        virtual_private=%v4:10.0.0.0/8,%v4:192.168.0.0/16,%v4:172.16.0.0/12,%v4:25.0.0.0/8,%v6:fd00::/8,%v6:fe80::/10
        oe=off
        protostack=netkey
        nhelpers=0
        interfaces=%defaultroute
conn openswan
        authby=secret
        type=tunnel
        auto=start

        left=10.100.3.8
        leftid=161.189.197.109
        leftsubnet=10.100.0.0/16
        leftnexthop=%defaultroute

        right=71.132.9.58
        rightsubnet=172.16.0.0/16
        rightnexthop=%defaultroute

        ike=aes256-sha1;modp1024
        ikelifetime=86400s
        phase2=esp
        phase2alg=aes256-sha1;modp1024
        salifetime=3600s
        pfs=yes
```
检查并配置/etc/ipsec.d/ipsec.secrets

```
161.189.197.109  71.132.9.58 : PSK "psk123"
```
重新启动IPsec服务

```
sudo systemctl restart ipsec
```
### 2.3 验证连通性

```
sudo systemctl status ipsec
```
预期输出

```
"openswan" #2: STATE_QUICK_I2: sent QI2, IPsec SA establis...ive}
```

登录到VPN实例，Ping对方VPN实例，验证VPN是否正常运行

```
# Ping VPN主实例
ping 10.100.3.8
PING 172.16.3.8 (172.16.3.8) 56(84) bytes of data.
64 bytes from 172.16.3.8: icmp_seq=1 ttl=64 time=31.3 ms
64 bytes from 172.16.3.8: icmp_seq=2 ttl=64 time=31.3 ms
64 bytes from 172.16.3.8: icmp_seq=3 ttl=64 time=31.3 ms
64 bytes from 172.16.3.8: icmp_seq=4 ttl=64 time=31.3 ms
```

##  3. 故障切换

### 3.1 模拟故障

在本地模拟环境的VPN实例上，执行如下命令模拟故障

```
sudo systemctl stop ipsec
```
执行如下命令，观察keepalived状态

```
sudo systemctl status keepalived -l
```
预期输出

```
2月 24 15:19:36 ip-10-100-3-8.cn-northwest-1.compute.internal Keepalived_vrrp[3336]: /usr/sbin/pidof  /usr/libexec/ipsec/pluto exited with status 1
2月 24 15:19:39 ip-10-100-3-8.cn-northwest-1.compute.internal Keepalived_vrrp[3336]: /usr/sbin/pidof  /usr/libexec/ipsec/pluto exited with status 1
2月 24 15:19:42 ip-10-100-3-8.cn-northwest-1.compute.internal Keepalived_vrrp[3336]: /usr/sbin/pidof  /usr/libexec/ipsec/pluto exited with status 1
2月 24 15:19:45 ip-10-100-3-8.cn-northwest-1.compute.internal Keepalived_vrrp[3336]: /usr/sbin/pidof  /usr/libexec/ipsec/pluto exited with status 1
2月 24 15:19:48 ip-10-100-3-8.cn-northwest-1.compute.internal Keepalived_vrrp[3336]: /usr/sbin/pidof  /usr/libexec/ipsec/pluto exited with status 1
2月 24 15:19:51 ip-10-100-3-8.cn-northwest-1.compute.internal Keepalived_vrrp[3336]: /usr/sbin/pidof  /usr/libexec/ipsec/pluto exited with status 1
2月 24 15:19:54 ip-10-100-3-8.cn-northwest-1.compute.internal Keepalived_vrrp[3336]: /usr/sbin/pidof  /usr/libexec/ipsec/pluto exited with status 1
2月 24 15:19:57 ip-10-100-3-8.cn-northwest-1.compute.internal Keepalived_vrrp[3336]: /usr/sbin/pidof  /usr/libexec/ipsec/pluto exited with status 1
2月 24 15:20:00 ip-10-100-3-8.cn-northwest-1.compute.internal Keepalived_vrrp[3336]: /usr/sbin/pidof  /usr/libexec/ipsec/pluto exited with status 1
```
### 3.2 验证切换是否成功

>目前的设定是每 3 秒检测一下ipsec进程是否存在，如果连续 10 次（30秒）故障仍旧存在，会触发 Master 节点及 Slave 节点的切换流程。

在控制台查看EIP是否完成漂移

![Alt text](https://github.com/toreydai/ha-vpn-aws-china/blob/main/images/ha_vpn_failover.png)

在本地模拟环境的VPN实例上，Ping备用实例的私有IP，验证是否切换成功

```
# Ping VPN备实例
ping 10.100.4.8
PING 10.100.4.8 (10.100.4.8) 56(84) bytes of data.
64 bytes from 10.100.4.8: icmp_seq=1 ttl=64 time=25.6 ms
64 bytes from 10.100.4.8: icmp_seq=2 ttl=64 time=25.6 ms
64 bytes from 10.100.4.8: icmp_seq=3 ttl=64 time=25.6 ms
```

