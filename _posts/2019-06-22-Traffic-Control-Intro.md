---
layout:     post
tags:
    - CDN
---
# Traffic Control Introduction  
参考<https://traffic-control-cdn.readthedocs.io/en/latest/overview/introduction.html>  
在Traffic Control的介绍中，第一话就是它是一个CDN的控制平面，一个有意思的话，如同通信网络协议中的七号信令，或者现在流行的service mesh的控制平面，都是同样的思想，官方介绍中的定义是：它由一组用于配置，管理，调度客户（请求）流量到http缓存代理服务器的应用构成。http缓存代理服务器即Cache服务器，既然Traffic Control是一个控制平面，那么可以理解clients, Cache Servers组成了CDN的数据平面。因此，理论上Traffic Control可以和任何一种http缓存代理服务器一起组成CDN系统，虽然现在Traffic Control现在是使用了[Apache Traffic Server](http://trafficserver.apache.org/)作为Cache Server。  

Traffic Control最初是由Comcast开发并用于内部使用，于2015年开源，现在是Apache的顶级项目。  

下图是Traffic Control的架构图：  
![img](https://traffic-control-cdn.readthedocs.io/en/latest/_images/traffic_control_overview_3.png)  

## Traffic Ops  
用于存储Cache Server,Delivery Services的配置，并且提供api让外部可以操作CDN数据，可以认为是Traffic Control这个CDN控制平面的一种实现中，核心的组件之一，主要负责了对CDN业务数据模型的管理。  
## Traffic Router  
即带了调度策略的DNS授权服务器，Traffic Control介绍的调度策略是结合了IP地理库和Cache Server的健康、容量、状态等信息来将客户请求调度到最佳Cache Server.  

## Traffic Monitor  
对Cache Server做健康监控、收集数据。这个组件是CDN控制平面、数据平面之间的一个变化点，通过在这个组件做合适的抽象、适配，就可以使得Traffic Control可以结合不同类型的Cache Server，而不局限在Apache Traffic Server.  

## Traffic Stats  
 用于收集、存储来自Cache Servers的实时数据，这些数据可以被Traffic Router用作调度策略的一个影响因素，使得流量负载均衡。  

 ## Traffic Portal  
 Traffic Portal is a web interface which uses the Traffic Ops API to present CDN data and the controls to manipulate it in a user-friendly interface.  

 ## Traffic Vault  
 Traffic Vault is used as a secure key/value store for SSL private keys used by other Traffic Control components.