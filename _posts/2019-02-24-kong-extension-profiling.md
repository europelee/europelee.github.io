# Kong 扩展分析(1)
## 简介
[Kong](https://konghq.com)是一个基于nginx,利用lua进行扩展的api gateway. 当然，随着service mesh的流行，Kong也增加相应的解决方案。我想基于之前的Kong扩展开发做一些分享系列，文章系列遵循从应用（跳过，这程度应该是最基本的，网上这类文章也比较泛滥）、到定制、再到实现探究，由表及内的过程。初步想法是首先是分享我之前所做的扩展工作及碰到的问题，对api gateway的设计思考（即本文），其次是探索Kong本身是如何基于nginx做扩展的和本身的扩展设计。  

首先介绍下扩展的两个需求authentication, and authorization, 即身份认证，和权限验证。通俗的说，一个是验证发起api请求的用户是否合法，另一个是验证该用户是否有权限访问该api. 熟悉Kong的同学应该知道Kong本身附带了多种关于authentication的[plugins](https://docs.konghq.com/hub/), authorization功能同样在Kong的Traffic Contorl的ACL插件找到对应，那么为什么还要自己定制呢，后面将会详细解释。  

然后将以代码的方式介绍主要扩展步骤，以及碰到的问题，最后介绍下我以为的api gateway。

## why custom authentication and authorization plugins  
先介绍下背景，我所在的部门或者公司有一个统一的账号及权限管理组件，你可以认为是轻量级的，类似AWS Identity and Access Management(IAM), 在我们的业务中，会过来向api gateway请求的用户都是来自注册到这个账号管理组件的，我想这时候熟悉Kong以及有此相似背景的同学已经想到了答案了。  
Kong的authentication, authorization插件的应用需要基于Kong内置的Consumer, group概念，而这些就和我们的IAM中特性相重叠，虽然Kong的实现也考虑到这一点，给Consumer了一个custom_id的属性，用来关联Kong用户的自己的账号库，但是这种设计并未带来多大的改观，更多的是一个补救措施，打上一个patch。双重的Consumer，权限组只会使得运营更加复杂，容易出错。  
基于上面的理由以及Kong的插件扩展机制，决定了authentication, authorization定制开发的合理性。  

## how implement, take authentication as example
基本上参考Kong的[plugins源代码](https://github.com/Kong/kong/tree/master/kong/plugins)，即可以把功能实现，主要是实现自己的代码逻辑。  
```lua
---[[ runs in the 'access_by_lua_block'
function Auth4IAMHandler:access(plugin_conf)
  Auth4IAMHandler.super.access(self)
  -- check if preflight request and whether it should be authenticated
  if not plugin_conf.run_on_preflight and get_method() == "OPTIONS" then
    return
  end
  -- check uri whitelist
  local is_white = check_uri_whitelist(ngx.var.request_uri, plugin_conf.uri_whitelist)
  if is_white then
    set_anonymous()
    return
  end
  local ok, err = do_authentication(plugin_conf)
  if not ok then
    return responses.send(err.status, {message=err.message})
  end
end --]]
```
这里说下plugin的api扩展比较麻烦点，这是因为Kong自带的curd_helpers.lua仅提供操作单个资源对象(resource)(可能由于Kong本身仅有这样的需求，或者因为Kong支持的其中一种数据库Cassandra没有像PostgreSQL那样的ACID能力), 但我们也需要操作资源对象列表(collection)，所以其实我们需要做面向对象的继承扩展，同时拥有collection, resource的操作能力。lua没有像java, python那样显示的继承语法，通过lua的元表可以达到目的。
```lua
local parent_crud    = require "kong.api.crud_helpers"
local acl_crud = {}
setmetatable(acl_crud, {__index = parent_crud})
```  

## api gateway design
架构设计本身只有各种tradeoff、合不合适，没有对错，在产品从demo, 原型再到GA, 以及业务需求从简单到复杂的不同阶段中，都有各种架构存在的合理性。Kong除了提供核心的路由功能，还提供了很多开箱即用的[插件](https://docs.konghq.com/hub/)功能, 但是这些很多都和Kong的内置对象consumer等做了绑定(即前面说的问题)，对于那些拥有自己的IAM组件的团队来说并不讨好。对于这些团队而言，api gateway仅需要提供这些功能的插件接口，作为一个轻量级的集成方，与IAM这些组件定好契约，通常实现的插件调用组件提供的REST接口，并且最好由这些组件提供类似DNS的TTL特性，这样api gateway可以进行缓存，提高api gateway的处理性能。