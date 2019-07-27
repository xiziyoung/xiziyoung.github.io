---
title: php递归和引用实现的无限极分类
date: 2018-07-12 18:10:47
updated: 2018-07-12 19:02:12
categories:
- PHP
tags:
- php
- algorithm
---
## 无限极分类
类似省、市、区,构建成一个父级下面有多个子元素的数据结构;子元素与父元素之间通常使用id和parentId来关联;   

### php递归实现无限极分类:   
该方法实现的无限极分类在数据量和子层级较多时,很慢,因为递归调用每次都要遍历大量的数据;    
代码示例:   
```php
<?php
function buildTreeByRecursion(array $elements, $rootValue = 0, $parentField = "parent", $index = '', $level = 0)
{
    $branch = array();
    $level++;
    foreach ($elements as $k => $element) {
        $element['_level'] = $level;
        if ($element[$parentField] == $rootValue) {
            unset($elements[$k]);//减轻下次递归时遍历的数量
            $children = buildTreeByRecursion($elements, $element['id'], $parentField, $index, $level);
            if ($children) {
                $element['children'] = $children;
            }

            if (empty($index)) {
                $branch[] = $element;
            } else {
                $branch[$element[$index]] = $element;
            }
        }
    }
    return $branch;
}
```

### php引用传值方式实现无限极分类    
首先我们看下简单php的引用传值:   
```php
<?php
$list = [
    0 => ['id' => 1,],
    1 => ['id' => 2,],
    2 => ['id' => 3,]
];

$tree = [
    0 =>  &$list[0],
    1 =>  &$list[1],
    2 =>  $list[2],
];
var_dump($tree);

$list[0]['children'] = ['first new child'];
$list[1]['children'] = ['second new child'];
$list[2]['children'] = ['third new child'];
var_dump($tree);
exit();
```
php的应用传值是将变量对应的内存地址指向同一个地址,即$a = &$b;   
那么修改$a或者$b的值,实际相当于修改的是他们在内存中的地址处的值, 那么两个变量的值都将变化,因为他们的地址指向的值变了; 

地址引用无限极分类:  
该方法很快, 因为不用像递归那样多次遍历数据;             
```php
<?php
function  getTreeByQuote($list, $pid = 0, $parentField = 'parent')
{
    $tree = [];
    if (!empty($list)) {
        //第一步 构造数据:修改为以id为下标的列表
        $newList = [];
        foreach ($list as $k => $v) {
            $newList[$v['id']] = $v;
        }

        //第二步: 遍历数据,生成树状结构
        foreach ($newList as $value) {
            if ($pid == $value[$parentField]) {
                $tree[] = &$newList[$value['id']];
            } elseif (isset($newList[$value[$parentField]])) {
                $newList[$value[$parentField]]['children'][] = &$newList[$value['id']];
            }
        }
    }
    return $tree;
}

```

**测试**:
```php
<?php
/**
* 前面的递归和引用实现无限极分类函数
 */

$data = file_get_contents('./category.json');
$data = json_decode($data, true);

$_base_time = time();
$_base = memory_get_usage();

//$res = buildTreeByRecursion($data, 0, 'parentId');
$res = getTreeByQuote($data, 0, 'parentId');

$_end_time = time();
$_end = memory_get_usage();
var_dump('内存使用'.(int)(($_end-$_base)/1024).'kb');   
var_dump('时间使用'.($_end_time-$_base_time).'s');
var_dump($res);exit();
echo json_encode($res,  JSON_UNESCAPED_UNICODE);exit();
```
测试数据: 4818条数据,其中一级44个,二级元素452个,三级元素4326个;   
递归方式,内存使用2082kb,时间使用11s;    
引用方式,内存使用608kb,时间使用0s;  