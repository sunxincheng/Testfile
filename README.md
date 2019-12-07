# Testfile
一、ubuntu 18.04配置PPPoE v6服务器需要安装如下安装包：

1、sudo apt install radvd

2、sudo apt install pppoe

其中radvd用于发送IPv6路由广告信息，pppoe用于拨号服务。

二、参数配置。

1、编辑/etc/ppp/pppoe-server-options配置文件，该配置文件用户设置pppoe服务参数。若需要提供IPv6地址则需要添加+ipv6选项。

require-chap
login
lcp-echo-interval 10
lcp-echo-failure 2
defaultroute
noipdefault
+ipv6

2、/etc/ppp/options配置文件使用默认值即可。

3、创建目录/etc/ppp/ipv6-radvd

4、创建pppoe ipv6启动脚本/etc/ppp/ipv6-up.d/radvd。

#!/bin/sh

ADDR=$(echo $PPP_REMOTE | cut -d : -f 3,4,5,6)

if test x$ADDR == x ; then
    echo "Unable to generate IPv6 address"
    exit 0
fi

ADDR=2001:470:8192:BEEF:$ADDR

#add route
route -6 add $ADDR/128 dev $PPP_IFACE
#generate radvd config
RAP=/etc/ppp/ipv6-radvd/$PPP_IFACE
RA=$RAP.conf

echo "interface $PPP_IFACE {" >> $RA
echo "\tAdvManagedFlag off;" >> $RA
echo "\tAdvOtherConfigFlag on;" >> $RA
echo "\tAdvSendAdvert on;" >> $RA
echo "\tMinRtrAdvInterval 5;" >> $RA
echo "\tMaxRtrAdvInterval 100;" >> $RA
echo "\tUnicastOnly on;" >> $RA
echo "\tAdvSourceLLAddress on;" >> $RA
echo "\tprefix 2001:470:8192:BEEF::/64 {};" >> $RA
echo "};" >> $RA

#start radvd
/usr/sbin/radvd -C $RA -p $RAP.pid

5、创建pppoe ipv6关闭脚本/etc/ppp/ipv6-down.d/radvd

#!/bin/sh

RAP=/etc/ppp/ipv6-radvd/$PPP_IFACE
kill `cat $RAP.pid` || true
kill `cat $RAP.dhcp.pid` || true
rm -f $RAP.*
ADDR=$(echo $PPP_REMOTE | cut -d : -f 3,4,5,6)
ADDR=2001:470:8192:BEEF:$ADDR
ARPA=$(ipv6_rev $ADDR)
nsupdate << EOF
    update delete $ARPA
    send
    update delete $PPP_IFACE.tunnel.ipv6.icybear.net
    send
EOF
exit 0

6、修改/etc/ppp/chap-secrets文件，创建pppoe拨号用户名和密码。

"test" * "test" *

三、pppoe服务器启动。

pppoe-server -I eth1

四、pppoe客户端启动。

1、修改客户端/etc/ppp/options文件支持ipv6地址。

+ipv6

2、pppoe客户端启动。

pppoe-setup

pppoe-connect
