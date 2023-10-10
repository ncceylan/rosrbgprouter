# 基本原理
- 利用unraid lxc特性，建立透明网关与DNS解析服务器，占用资源小，性能高，稳定性好
- 将透明网关生成的非本地路由表直接导入ROS，利用ROS通过路由表进行分流
- 非本地路由表直接由透明网关进行访问，由tun网卡进行流量转发，理论上，性能更好
- DNS选取easymosdns项目，解决dns污染与分流问题
# 特别感谢以下相关教程，排名不分先后
[使用RouterOS，OSPF 和OpenWRT给国内外 IP 分流](https://www.truenasscale.com/2021/12/13/195.html) 

[使用 RouterOS，OSPF 和树莓派为国内外 IP 智能分流](https://idndx.com/use-routeros-ospf-and-raspberry-pi-to-create-split-routing-for-different-ip-ranges/)

[linux下部署Clash+dashboard](https://parrotsec-cn.org/t/linux-clash-dashboard/5169)

[基于路由协议的ip分流(RouterOS为例)](https://www.chiphell.com/thread-2438228-1-1.html)
# 创建lxc容器模板
使用官方debian容器创建
### 容器配置文件
进入unriad控制台，进入/mnt/cache/lxc文件夹，修改对应的配置文件，添加以下内容
```
lxc.cgroup2.devices.allow = c 10:200 rwm
lxc.mount.entry = /dev/net dev/net none bind,create=dir
```
### 开启第三方登录
```
nano /etc/ssh/sshd_config
service ssh restart
```
### 常用软件安装
```
apt install zsh git vim curl -y
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```
### 添加未知命令提示工具
```
nano ~/.zshrc

. /etc/zsh_command_not_found
#在文件末尾添加以上内容

source ~/.zshrc
#配置生效
```
# 轻松搭建分流DNS服务器
我们右键刚才建立好的模板，完整复制一个lxc容器，网络地址修改为192.168.10.153
```

chmod +x dns.sh
./dns.sh


crontab -e


在文件末尾添加以下内容
0 5 * * * sudo truncate -s 0 /etc/mosdns/mosdns.log && /etc/mosdns/rules/update-cdn
#每天5点升级域名库并清除mosdns日志文件

```
## 配置文件修改
- 调整日志级别
- 开启缓存
- 添加统计插件
- 添加dns记录详细解析
- 开启api
```
sudo mosdns service restart
```
# 透明网关的创建与设置
```
chmod +x installclashmeta.sh

./insclashmeta.sh
至此，透明网关安装完成，我们接下来需要一份透明配置

######### 锚点 start #######
# proxy 相关
pr: &pr {type: select, proxies: [默认,香港,台湾,日本,新加坡,美国,其它地区,全部节点,自动选择,DIRECT]}

#这里是订阅更新和延迟测试相关的
p: &p {type: http, interval: 3600, health-check: {enable: true, url: http://www.google.com/generate_204, interval: 300}}

use: &use
  type: select
  use:
  - provider1
  - provider2

######### 锚点 end #######


# url 里填写自己的订阅,名称不能重复
proxy-providers:
  provider1:
    <<: *p
    url: ""

  provider2:
    <<: *p
    url: ""

mode: rule
ipv6: false
log-level: info
allow-lan: true
mixed-port: 7890
unified-delay: false
tcp-concurrent: true
external-controller: 0.0.0.0:9090
secret: '123456789'

geodata-mode: true
geox-url:
  geoip: "https://fastly.jsdelivr.net/gh/MetaCubeX/meta-rules-dat@release/geoip.dat"
  geosite: "https://fastly.jsdelivr.net/gh/MetaCubeX/meta-rules-dat@release/geosite.dat"
  mmdb: "https://fastly.jsdelivr.net/gh/MetaCubeX/meta-rules-dat@release/country.mmdb"

profile:
  store-selected: true
  store-fake-ip: true
  tracing: true

sniffer:
  enable: true
  sniff:
    TLS:
      ports: [443, 8443]
    HTTP:
      ports: [80, 8080-8880]
      override-destination: true
interface-name: eth0
tun:
  device: utun
  enable: true
  stack: system
  auto-route: true
  auto-detect-interface: false

dns:
  enable: true
  listen: :1053
  ipv6: false
  enhanced-mode: redir-host
  fake-ip-range: 28.0.0.1/8
  fake-ip-filter:
    - '*'
    - '+.lan'
    - '+.local'
  default-nameserver:
    - 192.168.10.9
  nameserver:
    - 192.168.10.9
  proxy-server-nameserver:
    - 192.168.10.9
  nameserver-policy:
    "geosite:cn,private":
      - 192.168.10.9

proxies:
  # - name: "WARP"
  #   type: wireguard
  #   server: engage.cloudflareclient.com
  #   port: 2408
  #   ip: "172.16.0.2/32"
  #   ipv6: "2606::1/128"        # 自行替换
  #   private-key: "private-key" # 自行替换
  #   public-key: "public-key"   # 自行替换
  #   udp: true
  #   reserved: "abba"           # 自行替换
  #   mtu: 1280
  #   dialer-proxy: "dns"
  #   remote-dns-resolve: true
  #   dns:
  #     - https://dns.cloudflare.com/dns-query

proxy-groups:

  - {name: 默认, type: select, proxies: [DIRECT, 香港, 台湾, 日本, 新加坡, 美国, 其它地区, 全部节点, 自动选择]}
# - {name: dns, type: select, proxies: [DIRECT, WARP, 香港, 台湾, 日本, 新加坡, 美国, 其它地区, 全部节点, 自动选择]}  # 加入 WARP  
#分隔,下面是地区分组
  - {name: 香港, <<: *use,filter: "(?i)港|hk|hongkong|hong kong"}

  - {name: 台湾, <<: *use, filter: "(?i)台|tw|taiwan"}

  - {name: 日本, <<: *use, filter: "(?i)日本|jp|japan"}

  - {name: 美国, <<: *use, filter: "(?i)美|us|unitedstates|united states"}

  - {name: 新加坡, <<: *use, filter: "(?i)(新|sg|singapore)"}

  - {name: 其它地区, <<: *use, filter: "(?i)^(?!.*(?:🇭🇰|🇯🇵|🇺🇸|🇸🇬|🇨🇳|港|hk|hongkong|台|tw|taiwan|日|jp|japan|新|sg|singapore|美|us|unitedstates)).*"}

  - {name: 全部节点, <<: *use}

  - {name: 自动选择, <<: *use, tolerance: 2, type: url-test}

rules:
  # - AND,(AND,(DST-PORT,443),(NETWORK,UDP)),(NOT,((GEOSITE,cn))),REJECT # quic

  - MATCH,默认

解压配置

tar -xvf ./clash.meta.tar

复制systemctl启动配置文件

mv ./clash.service /etc/systemd/system

配置防火墙iptables

apt-get install iptables -y

配置iptables


bash ./iptablesConfig


因为iptables重启后规则会丢失，所以需要开机重新导入规则。


apt install iptables-persistent -y

这里第一个提示的是IPV4的iptables配置的保存，第二个是V6的，看情况进行选择，因此教程只做v4 所以V6 就选NO 不保存。

检测是否完整保存配置。

cat /etc/iptables/rules.v4



至此，透明网关安装完成，我们接下来需要一份透明配置

```

## 路由地址宣告
修改bird2配置文件为以下内容 ，文件在/etc/bird/bird.conf
```
log syslog all;

router id 192.168.10.100;

protocol device {
        scan time 60;
}

protocol kernel {
        ipv4 {
              import none;
              export all;
        };
}

protocol static {
        ipv4;
        include "routes4.conf";
}

protocol bgp {
        local as 65531;
        neighbor 192.168.10.5 as 65530;
        source address 192.168.10.100;
        ipv4 {
                import none;
                export all;
        };
}
```
## 非本地ip表获取
```

cd /home

chmod +x iplist.sh
赋予权限

./iplist.sh

执行程序


crontab -e



0 5 * * * /bin/bash /home/iplist.sh

```
## 透明网关启动
```
sudo systemctl enable clash
#开机启动
sudo systemctl start clash
#开始启动



sudo systemctl status clash
#状态查看
```
# ros的设置
## 方式一
全局模式
```

/routing/bgp/connection
add name=clash local.role=ebgp remote.address=192.168.10.100 .as=65531 routing-table=main router-id=192.168.10.5 as=65530 multihop=yes
# 添加一个BGP连接，名称为clash，本地角色为ebgp，远程地址为192.168.10.100，自治系统号为65531，路由表为bypass，路由器ID为192.168.10.5，自治系统号为65530，启用多跳选项


/ip firewall mangle add action=accept chain=prerouting src-address=192.168.10.100
# 添加一个防火墙Mangle规则，动作为接受，链为prerouting，源地址为192.168.10.253

```
## 方式二
```

/ip route
add distance=1 gateway=pppoe-out1 routing-table=bypass comment=pass
# 添加一条路由规则，距离为1，网关为pppoe-out1，路由表为bypass，注释为pass

/routing/bgp/connection
add name=clash local.role=ebgp remote.address=192.168.10.100 .as=65531 routing-table=bypass router-id=192.168.10.5 as=65530 multihop=yes
# 添加一个BGP连接，名称为clash，本地角色为ebgp，远程地址为192.168.10.100，自治系统号为65531，路由表为bypass，路由器ID为192.168.10.5，自治系统号为65530，启用多跳选项

/ip firewall mangle add action=accept chain=prerouting src-address=192.168.10.100
# 添加一个防火墙Mangle规则，动作为接受，链为prerouting，源地址为192.168.10.253

/ip firewall address-list add list=proxy address=192.168.10.32
# 添加一个地址列表，名称为proxy，包含地址192.168.10.32


/ip firewall mangle add action=mark-routing chain=prerouting src-address-list=proxy dst-port=80,443 dst-address-type=!local protocol=tcp new-routing-mark=bypass
# 添加一个防火墙Mangle规则，动作为标记路由，链为prerouting，源地址列表为proxy，连接类型tcp。目标端口为80和443，目标地址类型不是本地地址，新的路由标记为bypass

重启路由
```
