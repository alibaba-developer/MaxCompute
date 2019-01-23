# MaxCompute studio与权限那些事儿
背景知识
MaxCompute拥有一套强大的安全体系，来保护项目空间里的数据安全。用户在使用MaxCompute时，应理解权限的一些基本概念：

- 权限可分解为三要素，即主体（用户账号或角色），客体（表/资源/函数等），以及操作（与特定客体类型相关），详细参考 https://help.aliyun.com/document_detail/27935.html。
- 授权有两种方式：ACL(基于对象，grant语句)和Policy(基于策略，policy file)。
- 跨项目授权使用package：https://help.aliyun.com/document_detail/34602.html。
- 可通过列标签实现表中列不同的访问控制：https://help.aliyun.com/document_detail/34604.html。
- 为了方便用户更好的理解与使用MaxCompute权限，studio实现了以下功能：

<h4>权限查看</h4>

用户在project下有哪些权限，可通过show grants语句获得。studio编辑器已集成权限相关的语句(https://help.aliyun.com/document_detail/27936.html) 通过快捷键(Windows: Ctrl + J , MAC: Command + J )唤出live template，然后搜索即可：
<div style="text-align:center" align="center">
<img src="/images/studio与权限.png" align="center" />
</div>

另外，studio对此也提供了图形化的方式显示用户的权限。如下图，点击工具栏上的show privileges按钮，弹出Show user privileges对话框，点击search button, 下方就会显示用户在该project下的权限：
<div style="text-align:center" align="center">
<img src="/images/studio与权限1.png" align="center" />
</div>

json标签页是所有权限的汇总，点击table标签页，则显示用户在table上的权限。鼠标悬停在table标签页上，则提示table的权限说明：
<div style="text-align:center" align="center">
<img src="/images/studio与权限2.png" align="center" />
</div>

<h4>权限异常诊断</h4>

当因缺少权限导致任务报鉴权失败异常时，可通过studio的权限异常诊断，快速寻找解决方案。如下图，点击工具栏上的权限异常诊断按钮，弹出权限异常诊断对话框，在上方文本框中输入完整的鉴权异常信息，然后点击ok按钮，则下方文本框会显示可能的解决方案：
<div style="text-align:center" align="center">
<img src="/images/studio与权限3.png" align="center" />
</div>

<h4>权限语句编写</h4>

MaxCompute提供了一系列的权限语句，studio SQL编辑器已集成这些语句，用户可以利用studio来执行这些语句以完成相应的权限操作。具体的，通过快捷键(Windows: Ctrl + J , MAC: Command + J )唤出live template，然后搜索：
<div style="text-align:center" align="center">
<img src="/images/studio与权限4.png" align="center" />
</div>
另外，在编写授权语句过程中，也支持相应的代码智能提示：
<div style="text-align:center" align="center">
<img src="/images/studio与权限5.png" align="center" />
</div>

<h4>授权语句生成</h4>

除了手写授权语句，studio也支持图形化给用户授权，点击工具栏上的show privileges按钮，弹出Show user privileges对话框，点击Grant privilege标签页，选择好授权对象，下方的SQL窗格就会同步显示其对应的授权语句，然后点击execute grant command，等待后台完成即可。
<div style="text-align:center" align="center">
<img src="/images/studio与权限6.png" align="center" />
</div>

<h4>studio中的权限</h4>

- 添加MaxCompute project时，studio会尝试列举project下的所有客体到本机，即用户必须有project的list权限。
- 显示表详情时，用户必须具备table的describe权限；显示自定义函数，则必须具备function的read权限。在编辑器中编写SQL，用到的table或function，则也必须有上述读权限。
- 在编辑器中运行某条SQL，则必须具备SQL中表的select权限，同时还必须有project的CreateInstance权限以能提交SQL任务。
- 开发好了UDF，要想发布，则必须有function的write权限。

<h4>权限好文</h4>

官方文档 https://help.aliyun.com/document_detail/27926.html

MaxCompute安全管理指南 https://yq.aliyun.com/articles/686800
