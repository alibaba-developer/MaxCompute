# 解决MaxCompute SQL count distinct多个字段的方法

按照惯性思维，统计一个字段去重后的条数我们的sql写起来如下：



Distinct的作用是用于从指定集合中消除重复的元组，经常和count搭档工作，语法如下

COUNT( { [ DISTINCT ] expression ] | * } )

这时，可能会碰到如下情况，你想统计同时有多列字段重复的数目，你可能会立马想到如下方法：

select count( distinct col1 , col2 , col3 , .......) from table

但是，这样是有问题的，如果值包含空，那么我们的结果是什么呢？如果你实验过，正如你实验的一样，结果会比实际少。

    	   
 	
   	
    	
<table>

<p style="text-align:center ">功课表</p>

<tr>

<th>a</th>

<td>b</td>

</tr>

<tr>

<th>1</th>

<td>null</td>

</tr>

<tr>

<th>2</th>

<td>x</td>

</tr>

<tr>

<th>1</th>

<td>null</td>

</tr>

</table>



count 结果为1；

因为MaxCompute count多列的时候，里面只要有一列为null，就忽略，不参加计算。

这个问题怎么解决？这里给出一个思路，先去重再count。

如，select count* from （select distinct a，b from table）

这样，count结果就和预期一致，结果为2；
