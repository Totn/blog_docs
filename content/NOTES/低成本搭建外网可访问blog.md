```json
{
    "date":"2023.02.07 15:38",
    "tags": ["BLOG","FRP"],
    "author": "Letotn"
}
```

作为程序员, 理应有一个自己的公开可以访问的博客, 同时可能有一些应用(比如`gitea`等代码托管平台)需要长时间运行；

但是买一个云服务器又有些浪费, 毕竟大部分时间服务器都无人访问, 现在各家云服务器厂商都是新用户大优惠而老用户不如狗, 运行一年之后再续的费用就上天了. 为了省钱决定用如下的方案来实现一个低成本博客:
- 家庭带宽
- 低功耗小主机
- frp内网空穿透实现公网访问
- 域名

在某鱼蹲了几天, 成功以300元入手一个矿渣主机: 锐角云, 配置N3450赛扬四核CPU, 8G内存, 64G emmc + 128g ssd硬盘; 待机功耗4w左右；惊喜的是原生支持`ubuntu`, 而且带WIFI和蓝牙, 安装好`ubuntu`之后再配置一下wifi密码可以直接使用了. 

### 折腾主机
锐角云主机自带win10系统, 到手之后直接用U盘重装成`ubuntu`;
本来应该是比较简单的事, 但在烧录`ubuntu`镜像到U盘这一步出点问题: 使用`rufus`在烧录前检查U盘坏块, 检完之后显示有坏块选择中止, 结果U盘写保护再也无法格式化.

搜索了许多办法都没有解决问题, 最后使用最彻底的办法: U盘量产工具.
- 使用`ChipGenius`检测U盘的主控芯片型号
- 按芯片型号搜索对应的U盘量产工具对其进行低格

我这U盘主控是Alcor, 找到一个Alcor全系列的量产工具, 终于把U盘救回来了.
当然,里面的数据都没有了.

后续就比较简单了, 使用`rufus`烧录镜像到U盘, 然后将小主机重启, 按F8选择U盘启动安装`ubuntu`.

### 配置WIFI
- 编辑配置文件:
  ```bash
  sodu vim /etc/netplan/00-installer-config-wifi.yaml
  ```
  增加以下内容:
  ```bash
  network:
  version: 2
  wifis:
    wlp2s0:
      access-points:
        "无线网络ssid":
          password: "wifi对应的密码"
      dhcp4:  true
  ```
- 使配置生效:
  ```bash
  # 检查语法，如有错误请检查缩进
  sudo netplan generate
  # 使配置生效
  sudo netplan apply
  ```

使用FRP实现内网穿透需要一个公网云服务器部署服务端, 好在网上已经有技术人搭建好了可免费转发FRP的服务器(感谢!, 赞美开源, 赞美共享精神!). 所以只需要配置好frpc程序, 将自己域名CNAME至指定域名, 我们的博客就可以正常访问了.

域名为原有域名续费, 价格150元续2年(域名过期了收到邮件提醒才想起自己有个域名), 配置好域名CNAME至FRP转发服务器, 绑定博客程序监听端口, 完工.

总花费为450元, 网络和电费可忽略不计. 相对每年都要续费的云服务器来说, 可以说是非常省钱了.