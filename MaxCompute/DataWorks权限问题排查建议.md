# MaxCompute/DataWorks权限问题排查建议
MaxCompute：计算引擎
MaxCompute底层计算引擎有自己的安全权限体系，包括ACL、Policy授权体系。具体可以了解：https://help.aliyun.com/document_detail/27924.html

<div style="text-align:center" align="center">
<img src="/images/DataWorks权限.png" align="center" />
</div>

DataWorks：数据工场
DataWorks为MaxCompute上层的云数仓开发工具，拥有自身的权限模型外还支持底层MaxCompute底层数据授权体系。具体详见：https://help.aliyun.com/document_detail/92594.html

查看MaxCompute上的角色
通过MaxCompute Console命令list roles;可以看到角色体系,role_开头都为DataWorks基于MaxCompute封装的角色及权限体系。介绍如下：

<div style="text-align:center" align="center">
<img src="/images/DataWorks权限2.png" align="center" />
</div>
<div style="text-align:center" align="center">
<img src="/images/DataWorks权限3.png" align="center" />
</div>

- _admin：__MaxCompute计算引擎的默认admin角色，可以访问项目空间内的所有对象、对用户或角色进行管理、对用户或角色进行授权。与项目空间 Owner 相比，admin 角色不能将 admin 权限指派给用户，不能设定项目空间的安全配置，不能修改项目空间的鉴权模型，admin 角色所对应的权限不能被修改。一般情况下，如未修改过权限，一般情况下admin角色用户只有一个为project owner账号。
odps@ clouder_bi>describe role admin;
```js
[users]
ALIYUN$***@aliyun-test.com
Authorization Type: Admin
```

MaxCompute project owner可以将admin角色授予其他子账号，用于进行MaxCompute底层的权限模型管理。

role_开头的角色也可以通过describe role 方式查看其角色所具备的权限点及角色里的用户列表。以开发者角色为例：

```js
odps@ clouder_bi>describe role role_project_dev;

[users]
RAM$yangyi.pt@aliyun-test.com:yangyitest

Authorization Type: Policy
A    projects/clouder_bi: *
A    projects/clouder_bi/instances/*: *
A    projects/clouder_bi/jobs/*: *
A    projects/clouder_bi/offlinemodels/*: *
A    projects/clouder_bi/packages/*: *
A    projects/clouder_bi/registration/functions/*: *
A    projects/clouder_bi/resources/*: *
A    projects/clouder_bi/tables/*: *
A    projects/clouder_bi/volumes/*: *
```

<h4>排查问题建议：</h4>
在普及完两个产品的权限体系之外，更多的用户会遇到各种权限的疑问或者问题。通常可以通过如下方式来排查：

- 首先，查看当前用户或指定用户所拥有的权限。
```js
show grants; --查看当前用户自己的访问权限
show grants for <username>; --查看指定用户的访问权限，仅由ProjectOwner和Admin才能有执行权限 。
show grants for RAM$主帐号:子帐号;
```

<div style="text-align:center" align="center">
<img src="/images/DataWorks权限4.png" align="center" />
</div>

可以看到用户所具有的角色及相关权限点。

查看指定对象的授权列表，一般获取表到人。
show acl for <objectName> [on type <objectType>];--查看指定对象上的用户和角色授权列表
支持的objecTtype: PROJECT, TABLE, JOB, VOLUME, INSTANCE, RESOURCE, FUNCTION,PACKAGE,TOPOLOGY,MATRIX,XFLOW,OFFLINEMODEL,STREAMJOB

<div style="text-align:center" align="center">
<img src="/images/DataWorks权限5.png" align="center" />
</div>

- 查看ACL是否生效（常常发生在授权之后返回OK，但是权限校验还是失败）
show SecurityConfiguration;--查看项目空间的安全配置

<div style="text-align:center" align="center">
<img src="/images/DataWorks权限6.png" align="center" />
</div>

除了通过命令行方式，也可以通过__++DataWorks>项目管理>MaxCompute高级配置++__里的ACL开关来确认是否打开。

<h4>Policy授权的查询</h4>
policy授权一般常见有两种，一个是项目级别的，一个是role级别的。

get policy;--获取项目级别的policy配置；
get policy on role <rolename>;--获取指定的role policy设置。

<div style="text-align:center" align="center">
<img src="/images/DataWorks权限7.png" align="center" />
</div>
