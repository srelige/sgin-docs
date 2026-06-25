# 第 11 章：权限后台不能只会新增

第二周一早，张三把运维平台的用户管理页面接到 sgin。

登录、创建用户、创建角色、绑定权限都能跑。可他很快发现一个尴尬问题：页面能把数据加进去，却很难维护。

用户多选了一个组，前端想保存最终勾选结果，却只能继续追加绑定。角色名字写错了，没有修改接口。菜单建错了，没有删除接口。路由权限需要手填 path，填错以后也不好修。

李四看完接口调用表，说：“这不是权限模型缺东西，是管理闭环没补齐。”

## 只会创建不叫后台

初始化接口能让系统跑起来，维护接口才能让系统长期可用。

真正的权限后台每天面对的是维护：

- 用户要分页查询。
- 组、角色、权限点要能搜索。
- 菜单要能改名、禁用、删除。
- 路由权限要能按 method/path 查。
- 多选框保存时要覆盖最终关系。
- 配错的绑定要能解绑。
- 刷新页面后要能重新拿到当前用户权限快照。

张三说：“这些不是业务模型，都是权限后台自己的日常动作。”

李四点头：“所以它们应该由内置管理接口补齐，而不是让每个前端自己绕。”

## 管理接口从配置路径开始

张三先看配置：

```txt
admin.path = /sgin-admin
```

李四说，下面的例子都按这个路径展开。实际项目如果把路径配置成 `/admin`，那接口前缀也跟着变。

查看当前权限管理状态：

```txt
GET /sgin-admin/state
```

查看当前登录用户权限快照：

```txt
GET /sgin-admin/me
```

扫描当前 Gin 路由：

```txt
GET /sgin-admin/routes
```

张三记住了：不要在前端写死一套和配置不一致的地址。页面应该从项目配置或部署约定里拿到管理入口。

## 标准维护接口

张三把用户管理页拆成几个常见动作：列表、详情、新增、修改、删除、启停。

李四说，这就是标准 CRUD。

内置管理接口对这些对象都提供维护能力：

```txt
users
groups
roles
permissions
menus
route-permissions
```

以用户为例：

```txt
GET    /sgin-admin/users
GET    /sgin-admin/users/:id
POST   /sgin-admin/users
PATCH  /sgin-admin/users/:id
DELETE /sgin-admin/users/:id
PATCH  /sgin-admin/users/:id/enabled
```

用户列表支持分页、用户名搜索和启用状态过滤。返回里会带用户所属组和角色，前端不必每行再查一遍。

组、角色和权限点列表也支持分页和关键字搜索。

菜单列表默认返回树。需要普通列表时，可以传 `tree=false`。

路由权限列表可以按 method、path、permission_code 和 enabled 过滤。

## 覆盖式关系保存

张三的页面里，用户所属组是一个多选框。

他原来想遍历新增项和删除项，分别调追加和解绑接口。李四让他停下。

前端最自然的动作是保存最终结果：

```txt
PUT /sgin-admin/users/:id/groups
PUT /sgin-admin/users/:id/roles
PUT /sgin-admin/groups/:id/roles
PUT /sgin-admin/roles/:id/permissions
```

请求体只需要表达最终 ID 集合：

```txt
ids = [1, 2, 3]
```

服务端在事务里清理旧关系，再写入新关系。

同一份 ids 提交多次，结果应该一致。传空数组，就是清空关系。

李四说：“多选框是覆盖式保存，不是让前端推理差量。”

## 解绑接口什么时候用

覆盖式接口已经能解决多数表单保存问题。

但有些页面是关系列表，每行一个“移除”按钮。这种场景用单条解绑更直接：

```txt
DELETE /sgin-admin/users/:id/groups/:group_id
DELETE /sgin-admin/users/:id/roles/:role_id
DELETE /sgin-admin/groups/:id/roles/:role_id
DELETE /sgin-admin/roles/:id/permissions/:permission_id
```

李四的判断很简单：

- 表单多选保存，用 `PUT` 覆盖最终集合。
- 单行移除按钮，用 `DELETE` 删除一条关系。

不要让前端为了一个常见交互写复杂同步逻辑。

## 当前用户权限快照

张三问：“登录响应已经有 menus 和 permissions，为什么还要一个 me 接口？”

李四说，刷新页面时前端可能只剩 token。

这时它需要重新知道：

- 当前用户是谁。
- 属于哪些组。
- 有哪些角色。
- 拥有哪些权限点。
- 能看到哪些菜单。

所以管理接口提供：

```txt
GET /sgin-admin/me
```

这个接口和登录响应一样，是前端渲染用的权限快照。

它不是后端安全边界。接口能不能执行，仍然要看 JWT、`LoadAccess()`、`RequireRoutePermission()` 或业务自己的权限逻辑。

如果前端只想判断管理员能力，当前约定很简单：登录用户的 groups 里包含 `admin`，就按管理员处理。

## 路由扫描和同步

动态路由权限最容易填错的是 path。

张三手填过一次：

```txt
GET /users
```

实际路由却是：

```txt
GET /system/users
```

页面看起来配置了权限，后端永远匹配不上。

李四让他直接从 Gin engine 扫描：

```txt
GET /sgin-admin/routes
```

返回当前应用真实注册的 method、path、handler，以及已有路由权限上的 permission_code。

如果要先把路由权限表补齐，可以调用：

```txt
POST /sgin-admin/route-permissions/sync
```

同步只补缺失的 method/path，不覆盖已经配置过的权限点。

新补出来的记录默认是禁用的，并且 permission_code 为空。它们只是后台页面上的待配置占位，不会立刻参与 `RequireRoutePermission` 放行判断。

## 删除要有保护

张三问：“删除是不是直接硬删？”

李四说，可以硬删，但必须保护关键关系。

内置 admin 用户不能删除。当前登录用户不能删除自己。

内置 admin 组和 admin 角色不能删除，也不能改名。

权限点如果已经被角色、菜单或路由权限引用，删除时会返回冲突。

父菜单下面还有子菜单时，也不能直接删除。

这些保护不是为了让后台复杂，而是避免一个误操作把权限系统拆坏。

## 张三的笔记

张三写下：

权限后台不能只提供创建接口，还要有列表、详情、修改、删除、覆盖式关系保存、解绑、当前用户快照、路由扫描和同步。前端负责界面体验，后端负责维护闭环和安全底线。