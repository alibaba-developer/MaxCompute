# MaxCompute SQL随机抽取N行数据
本文将为您介绍如何对数据随机取出数据的前 N 条数据。
<h4>示例数据</h4>
目前的数据，如下表所示：

<div style="text-align:center" align="center">
<img src="/images/SQL随机抽取N行数据.png" align="center" />
</div>

<h4>实现方法</h4>

通过order by rand()来实现随机抽取效果。


```js
 SELECT empno
  , ename
  , sal
  , job
  FROM emp
order by rand() limit 3

```
