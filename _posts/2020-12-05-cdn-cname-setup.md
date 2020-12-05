# CDN CNAME Setup

## **简介**

继[CDN Play](https://europelee.github.io/2020/11/14/cdn-play/), 我们来介绍下当前接入CDN厂商的方式，一如以前，我们以场景介绍作为出发点。

## **场景**

我们在A云有一个注册的域名 abc.net，其中使用了blog.abc.net写blog，并且将网站部署在了海外，现在我们让它成为Eason的第一个用户，境内访问加速。

## **步骤**

|  No.   | 对象  | 操作 |
|  ----  | ----  | ---- |
| 1  | pdns | 修改 zone.yaml,services下添加:<br>blog.abc.net.gslb.abc.site: ['%co.%cn.geo.abc.site', 'unknown.%cn.geo.abc.site', 'unknown.unknown.geo.abc.site'] |
| 2  | ats  | 添加加速域名,及其回源地址 |
| 3  | client | A云上面配置 blog.abc.net cname记录为blog.abc.net.gslb.abc.site| 

## **验证**

使用[站长工具](http://tool.chinaz.com), 浏览器输入 <http://ping.chinaz.com/blog.abc.net>, 可以发现返回的IP为Eason的cache IP, 或者直接访问blog.abc.net验证, 会发现摘要里面的地址为Eason的cache IP。
