
操作系统版本：Debian 12\.5\_x64


kamailio版本：5\.8\.2


kamailio作为专业的SIP服务器，可承担注册服务器的角色。今天记录下kamailio作为注册服务器，承接分机注册，并实现相互拨打的过程。


我将从以下几个方面展开：


* 模块配置
* 分机账号添加
* 无rtp代理的分机互拨
* 带rtp代理的分机互拨
* 配套资源下载


# **一、模块加载**


模块文档地址：


[https://kamailio.org/docs/modules/5\.8\.x/modules/registrar.html](https://github.com)


模块加载及配置（使用kamailio的默认配置即可）：




```
loadmodule "registrar.so" 

# ----- registrar params -----
modparam("registrar", "method_filtering", 1)
/* uncomment the next line to disable parallel forking via location */
# modparam("registrar", "append_branches", 0)
/* uncomment the next line not to allow more than 10 contacts per AOR */
# modparam("registrar", "max_contacts", 10)
/* max value for expires of registrations */
modparam("registrar", "max_expires", 3600)
/* set it to 1 to enable GRUU */
modparam("registrar", "gruu_enabled", 0)
```


kamailio的安装可参考这篇文章：


[kamailio5\.8安装及配置](https://github.com):[milou加速器](https://xinminxuehui.org)


# 二、添加分机


命令格式如下：




```
kamctl add   
```


示例：




```
kamctl add 2001 123456
```


如果提示如下错误：




```
ERROR: domain unknown: use usernames with domain or set default domain in SIP_DOMAIN
```


![](https://img2024.cnblogs.com/blog/300959/202409/300959-20240902222034494-688786626.png)


 则需要编辑kamctlrc文件，配置SIP\_DOMAIN变量。


![](https://img2024.cnblogs.com/blog/300959/202409/300959-20240902222048065-1077859506.png)


 账号信息存储在subscriber表中，效果如下：


![](https://img2024.cnblogs.com/blog/300959/202409/300959-20240902222103415-1101246539.png)


 可使用软电话注册验证，效果如下：


![](https://img2024.cnblogs.com/blog/300959/202409/300959-20240902222116618-1428287515.png)


 可添加多个用户：




```
kamctl add 2001 123456
kamctl add 2002 123456
```


数据表效果如下：


![](https://img2024.cnblogs.com/blog/300959/202409/300959-20240902222150457-421158568.png)


# 三、分机互拨


这里以内网的场景进行模拟。


分机2001和分机2002注册地址：192\.168\.137\.1


kamailio地址：192\.168\.137\.71


呼叫模型如下：


![](https://img2024.cnblogs.com/blog/300959/202409/300959-20240902222226579-1697174469.png)


## 1、存储注册信息


配置相关模块，示例如下：




```
loadmodule "db_mysql.so"
loadmodule "usrloc.so"

modparam("usrloc", "db_url", "mysql://root:test_123@192.168.137.1/kamailio71")
modparam("usrloc", "db_mode", 1)
modparam("usrloc", "use_domain", MULTIDOMAIN)
```


这里db\_mode用1，直接写数据库，也可以用2，定时刷新到数据库。


![](https://img2024.cnblogs.com/blog/300959/202409/300959-20240902222300610-1675161521.png)


## 2、添加分机路由


可以使用默认的LOCATION路由，也可以自定义路由，这里以自定义路由为例进行演示。


配置相关模块，示例如下：




```
loadmodule "sqlops.so"

modparam("sqlops","sqlcon","db=>mysql://root:test_123@192.168.137.1/kamailio71")
```


sqlops是用于查询数据库的"db\=\>"是定义的标识，后续查询时用db标签。


添加路由，使用 sql\_xquery 函数查询数据库；


route(EXTEN);


路由内容如下：




```
route[EXTEN] {
    if(!is_method("INVITE"))
        return;
    xlog("L_INFO","rU is $rU \n");

    if (!registered("location")) {
        xlog("L_INFO","$rU not registered\n");
        exit;
    }
    sql_xquery("db","SELECT contact FROM location WHERE username='$rU';","rb");
    if ( $xavp(rb=>contact) == $null) {
        xlog("L_INFO","contact is  null  \n");
    } else {
        xlog("L_INFO","contact is   $xavp(rb=>contact)  \n");
        $du =  $xavp(rb=>contact);
    }
    xlog("L_INFO","rU is $rU , du is $du\n");

    route(RELAY);
    exit;
}
```


配置截图如下：



![](https://img2024.cnblogs.com/blog/300959/202409/300959-20240902222359518-532992379.png)


## **3、呼叫测试**


这里使用分机2002拨打分机2001，路由成功，效果如下：



![](https://img2024.cnblogs.com/blog/300959/202409/300959-20240902222425664-629374668.png)


 配套配置文件可从如下渠道获取：


关注微信公众号（聊聊博文，文末可扫码）后回复 20240902 获取。


这里没有进行rtp代理，sdp里面写的是话机的ip地址：



![](https://img2024.cnblogs.com/blog/300959/202409/300959-20240902222545896-1404494805.png)


 配套的抓包文件可从如下渠道获取：


关注微信公众号（聊聊博文，文末可扫码）后回复 20240902 获取。



# 四、添加rtp代理


上述步骤中，实现了sip信令代理，如果要代理rtp数据，可以使用rtpproxy或rtpengine实现，这里以rtpengine为例进行示例。


呼叫模型如下：


![](https://img2024.cnblogs.com/blog/300959/202409/300959-20240902222634025-43227541.png)


## 1、安装rtpengine


可通过apt安装：




```
apt install rtpengine
```


也可通过源码安装，具体参考：


[debian10环境安装rtpengine](https://github.com)


默认配置文件：


/etc/rtpengine/rtpengine.conf


启动rtpengine：




```
systemctl start rtpengine
```


效果如下：


![](https://img2024.cnblogs.com/blog/300959/202409/300959-20240902222752993-402216450.png)


## 2、配置kamailio


文件：kamailio.cfg


加载模块，并配置参数：





```
loadmodule "rtpengine.so"

modparam("rtpengine", "rtpengine_sock", "udp:127.0.0.1:2223")
```


如果使用默认配置文件的规则（kamailio.cfg），需要先定义变量：




```
#!define WITH_NAT                                                                            
#!define WITH_RTPENGINE
```


配置效果如下：


![](https://img2024.cnblogs.com/blog/300959/202409/300959-20240902222842545-1306482894.png)


 配置路由（在NATMANAGE中，默认配置）：




```
 if(nat_uac_test("8")) {
        rtpengine_manage("SIP-source-address replace-origin replace-session-connection");
    } else {
        rtpengine_manage("replace-origin replace-session-connection");
    }
```


也可使用自定义路由（在NATMANAGE中）：




```
if (has_body("application/sdp")) {
    rtpengine_manage("trust-address replace-origin replace-session-connection");
}
```


这里以默认配置为例进行演示，配置如下：


![](https://img2024.cnblogs.com/blog/300959/202409/300959-20240902222927217-1206218067.png)


 重启kamailio服务生效：




```
systemctl restart kamailio
```


配套配置文件可从如下渠道获取：


关注微信公众号（聊聊博文，文末可扫码）后回复 20240902 获取。


## 3、使用rtpengine进行代理


使用2002呼叫2001，并在kamailio上进行抓包。


呼叫效果如下：



![](https://img2024.cnblogs.com/blog/300959/202409/300959-20240902223016209-2025917805.png)


 使用rtpengine进行rtp代理后，sdp里面写的是kamailio的ip地址：


![](https://img2024.cnblogs.com/blog/300959/202409/300959-20240902223029889-909143953.png)


 可使用wireshark播放rtp音频流：


![](https://img2024.cnblogs.com/blog/300959/202409/300959-20240902223044119-1150219687.png)


 配套抓包文件可从文末提供的渠道获取。


# **五、资源下载**


本文涉及源码及相关文件，可从如下途径获取：


关注微信公众号（聊聊博文，文末可扫码）后回复 20240902 获取。


![](https://img2024.cnblogs.com/blog/300959/202409/300959-20240902223112677-1025670615.png)


 好，就这么多了，别忘了点赞哈！


