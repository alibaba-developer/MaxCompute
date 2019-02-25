# maxcompute 2.0复杂数据类型之array

<h4>1. 含义</h4>

类似于Java中的array。有序、可重复。

<h4>2. 场景</h4>

什么样的数据，适合使用array类型来存储呢？这里列举了几个我在开发中实际用到的场景。

<h4>2.1 标签类的数据</h4>

为什么说标签类数据适合使用array类型呢？

（1）标签一般是一个只有key、没有value的结构；

（2）标签的数量（枚举值个数）会非常多；

（3）标签的变化会比较频繁；

（4）标签会过期；

因此，比起“创建多个字段”、“使用指定分隔符分隔的字符串”、“使用map”等方法，使用array是更合适的。

<h4>2.2 对象列表</h4>

对象有多种固定的属性，简单的key-value格式无法满足，可以使用array嵌套struct的方式定义。减少了维护数据字典的工作量。

<h4>3. 玩转array</h4>

3.1 数组字段拆成多行

3.1.1 explode

```js
select explode(t.arr) from (select array('a','b','c') as arr) t;
col
a
b
c
```

```js
select t1.id,t2.arr from (select 'xxx' as id,array('a','b','c') as arr) t1 lateral view explode(t1.arr) t2 as arr ;
id	arr
xxx	a
xxx	b
xxx	c
```

<h4>3.1.2 posexplode</h4>

```js
select posexplode(t.arr) from (select array('a','b','c') as arr) t;
pos	val
0	a
1	b
2	c
```

```js
select t1.id,t2.serialno,t2.arr from (select 'xxx' as id,array('a','b','c') as arr) t1 lateral view posexplode(t1.arr) t2 as serialno,arr ;
id	serialno	arr
xxx	0	a
xxx	1	b
xxx	2	c
```

<h4>3.2 多行合并成数组</h4>

3.2.1 不去重

```js
select collect_list(t.c1) as arr from ( select 'a' as c1 union all select 'a' as c1 union all select 'b' as c1) t;
arr
["a","a","b"]
```

<h4>3.2.2 去重</h4>

```js
select collect_set(t.c1) as arr from ( select 'a' as c1 union all select 'a' as c1 union all select 'b' as c1) t;
arr
["a","b"]
```

<h4>3.3 数组拼成字符串</h4>

```js
select concat_ws(',',t.arr) from (select array('a','b','c') as arr) t;
_c0
a,b,c
```

<h4>3.4 字符串转成数组</h4>

```js
select split('a,b,c',',');
_c0
["a","b","c"]
```

<h4>3.5 构造数组</h4>

```js
select array('aa','bb','cc');
_c0
["aa","bb","cc"]
```

<h4>3.6 数组元素排序</h4>

```js
select sort_array(array('b','c','e','a','d'));
_c0
["a","b","c","d","e"]
select sort_array(array(1,10,100,2,3));
_c0
[1,2,3,10,100]
```

<h4>3.7 数组中增加一项</h4>

```js
select split(concat('d,',concat_ws(',',t.arr)),',') as arr from (select array('a','b','c') as arr) t;
arr
["d","a","b","c"]
```

<h4>4. 常见用法</h4>

4.1 代替无法使用的with cube

例如现在有张下单记录流水表，记录着每一条下单记录，包含字段“订单ID”、“下单人ID”、“下单渠道(网站/app)”。

现在要统计“各渠道的下单人数和订单数”，渠道维度包含“不限”、“网站”、“APP”三项。

一般做这些包含“不限”的维度的聚合计算时，都使用group by xxx with cube关键字。但是maxcompute中暂时还不支持这个关键字，所以我们换另一种方法来实现。

```js
SELECT tt.`下单渠道`, COUNT(1) AS `下单人数`, SUM(tt.`下单量`) AS `下单量`
FROM (
    SELECT t1.`下单人ID`, t2.`下单渠道`, SUM(t1.`下单量`) AS `下单量`
    FROM (
        SELECT t.`下单人ID`, t.`下单渠道`, SUM(t.`下单量`) AS `下单量`
        FROM (
            SELECT `订单ID`, `下单人ID`, `下单渠道`, 1 AS `下单量`
            FROM `下单记录流水表`
        ) t
        GROUP BY t.`下单人ID`, 
            t.`下单渠道`
    ) t1
        LATERAL VIEW EXPLODE(array(t1.`下单渠道`, '不限')) t2 AS `下单渠道`
    GROUP BY t1.`下单人ID`, 
        t2.`下单渠道`
) tt
GROUP BY tt.`下单渠道`
```

<h4>4.2 数组是否相等</h4>
数组的相等或不等，无法通过“=”来判断，因此要尝试一些其他的方法。最常用的办法，就是转成字符串再比较。

<h4>4.2.1 考虑顺序是否一致</h4>
直接转成字符串后，比较是否相等

<h4>4.2.2 不考虑顺序是否一致</h4>
先排序，再转成字符串，然后比较是否相等
