---
layout:     post
title:      "日常开发必备工具箱"
tags:
    - Linux Mint
    - 开发工具
---
# 日常开发必备工具箱

## **简介**
性能普通的办公电脑和吃硬件的Window系统的冲突，加上日常开发及geek需要，让我在几年前就把厂里电脑的开发桌面环境迁移到linux,因为一直都在接触CentOS, 所以就出于方便、熟悉把个人开发电脑也装了CentOS用作开发桌面环境。  

只能说开弓无回头箭，开发活动不仅仅只是写代码，还会有IM工具沟通、视频会议、线下文档，打印等等需求，最大的问题是企业微信的安装，CentOS对Wine安装很不友好，最后只能在CentOS系统上用VirtualBox安装了deepin系统😄，再在deepin安装了一个企业微信，CentOS本质是用于服务器的系统，对个人使用不友好。  

现在乘着换电脑之际，捣鼓了[Linux Mint](https://www.linuxmint.com)系统，使用deepin-wine-for-ubuntu把企业微信给装上了，虽然是比较旧的版本，但可以满足日常沟通需要，免去了再搞个虚拟机系统的麻烦，参考这两链接就可以搞定了：https://zhuanlan.zhihu.com/p/363419059， https://blog.csdn.net/liu19721018/article/details/109689729

## **工具清单**
幸好像文档之类都是在线上Google Doc, Confluence管理，而电脑可以安装WPS用于偶尔线下需要，这也免去了之前的一顿骚操作：上传到Google Doc, 再在Google Doc编辑，最后再另存为下载到本地😅。  
下面列下常用或小众的工具(像WPS不在此列)，后续计划搞一个脚本自动安装。  

|type |name |what|install|comment |
|------|----------------|----------------|----------------|---------------------|
|开发|Visual Studio Code|码<br>Git查看<br>setting sync etc|微软官方网站直接下载|若下载慢，直接百度下，使用azure加速域名替换下载url|
|开发|Docker Desktop for Linux|构建程序开发、测试环境|官方网站|新鲜出炉|
|开发|Git|代码版本管理|apt-get|-|
|开发|Python,Go|语言包|官方|参考[pyenv](https://blog.csdn.net/inke88/article/details/59761696)|
|开发|zsh|好用的命令解释器|官方|参考[install ohmyzsh](https://gist.github.com/europelee/128c5fecedb71bd3fd4c95c3cda35bb1)|
|网络|shadowsocks,privoxy|翻墙|[部署](https://gist.github.com/europelee/f5681ae12e601985878ae314039617ab)|-|
|网络|SwitchyOmega|Chrome浏览器翻墙|[部署](https://github.com/FelisCatus/SwitchyOmega)|-|
|文档|XMind|思考|[下载](https://www.xmind.cn/download/)|-|
|IM|企业微信|沟通|deepin-wine|参考上面链接|
