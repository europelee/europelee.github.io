# Kong 扩展分析(1)
## 简介
[Kong](https://konghq.com)是一个基于nginx,利用lua进行扩展的api gateway. 当然，随着service mesh的流行，Kong也增加相应的解决方案。我想基于之前的Kong扩展开发做一些分享系列，文章系列遵循从应用（跳过，这程度应该是最基本的，网上这类文章也比较泛滥）、到定制、再到实现探究，由表及内的过程。初步想法是首先是分享我之前所做的扩展工作及碰到的问题，对api gateway的设计思考（即本文），其次是探索Kong本身是如何基于nginx做扩展的和本身的扩展设计。  

首先介绍下扩展的两个需求authentication, and authorization, 即身份认证，和权限验证。通俗的说，一个是验证发起api请求的用户是否合法，另一个是验证该用户是否有权限访问该api. 熟悉Kong的同学应该知道Kong本身附带了多种关于authentication的[plugins](https://docs.konghq.com/hub/), authorization功能同样在Kong的Traffic Contorl的ACL插件找到对应，那么为什么还要自己定制呢，后面将会详细解释。  

然后将以代码的方式介绍主要扩展步骤，以及碰到的问题，最后介绍下我以为的api gateway。

## why custom authentication and authorization plugins  
先介绍下背景，我所在的部门或者公司有一个统一的账号及权限管理组件，你可以认为是轻量级的，类似AWS Identity and Access Management(IAM), 在我们的业务中，会过来向api gateway请求的用户都是来自注册到这个账号管理组件的，我想这时候熟悉Kong以及有此相似背景的同学已经想到了答案了。  
Kong的authentication, authorization插件的应用需要基于Kong内置的Consumer, group概念，而这些就和我们的IAM中特性相重叠，虽然Kong的实现也考虑到这一点，给Consumer了一个custom_id的属性，用来关联Kong用户的自己的账号库，但是这种设计并未带来多大的改观，更多的是一个补救措施，打上一个patch。双重的Consumer，权限组只会使得运营更加复杂，容易出错。  
基于上面的理由以及Kong的插件扩展机制，决定了authentication, authorization定制开发的合理性。  
