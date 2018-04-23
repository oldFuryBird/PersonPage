---
layout: post
title:  "关于数组排序的讨论"
date:   2018-04-23 12:10:58 +0800
categories: php
tags: 工具
---
# 数组的排序
**问题: 如何对一个二维数组进行排序?**

- 实现通过二维的一个字段进行排序
- 实现像sql一样排序,order by o1,o2,o3....,依次用o1进行排序,如果排序完成之后,再按o2的排序


### php文档介绍 php的排序是基于快速排序实现,这里只讨论php的函数以及包的应用
```php
  # 要排序的数组
  $arr = [
      ['id'=>2,'name'=>'e','age'=>17],
      ['id'=>4,'name'=>'a','age'=>15],
      ['id'=>3,'name'=>'b','age'=>30],
      ['id'=>6,'name'=>'f','age'=>18],
      ['id'=>8,'name'=>'a','age'=>18],
  ];
  # 结果
  # 按id 升序排后的结果
  $arr_sort_by_id = [
    ['id'=>2,'name'=>'e','age'=>17],
    ['id'=>3,'name'=>'b','age'=>30],
    ['id'=>4,'name'=>'a','age'=>15],
    ['id'=>6,'name'=>'f','age'=>18],
    ['id'=>8,'name'=>'a','age'=>18],
  ];
  # 按 age 升序排后的结果
  $arr_sort_by_age = [
    ['id'=>4,'name'=>'a','age'=>15],
    ['id'=>2,'name'=>'e','age'=>17],
    ['id'=>6,'name'=>'f','age'=>18],
    ['id'=>8,'name'=>'a','age'=>18],
    ['id'=>3,'name'=>'b','age'=>30],
  ];
  # 先按age排 然后再根据id 降序 把排age中重复的
  $arr_sort_by_age_and_id = [
    ['id'=>4,'name'=>'a','age'=>15],
    ['id'=>2,'name'=>'e','age'=>17],
    ['id'=>8,'name'=>'a','age'=>18],
    ['id'=>6,'name'=>'f','age'=>18],
    ['id'=>3,'name'=>'b','age'=>30],
  ];

```
## php数组排序的方法
  - array_multisort 排序多个数组,功能强大
  - usort 按用户自定义的函数排序
  - uksort 按用户自定义函数比较键名排序

### usort 自定义函数
```php
    function orderBy( $key,$order = SORT_ASC ) {
      return function ( $a, $b ) use( $key, $order ){
          if($order == SORT_ASC)  return $a[$key]>$b[$key];
          if($order == SORT_DESC) return $a[$key]<$b[$key];
      };
    }

    $test_id  = $arr; //数组不是对象,数组是按值传递 与js不同
    $test_age = $arr;
    usort($test_id, orderBy('id'));
    usort($test_age, orderBy('age'));
    var_dump($test_id === $arr); //false
    var_dump($test_id === $arr_sort_by_id); // true
    var_dump($test_age === $arr_sort_by_age); // true
```

#### usort 很好的解决了 order by 1个字段的排序,但如果面对多个字段将变得复杂
  1. 查找age的重复项数组
  2. 将重复的数组提取出来 保留原索引键,并且按id逆序排序
  3. 如果每一个字段都需要指定正序,倒序或者字符串的排序规则等,那么就会更加增加复杂度

```php
function orderByKeys(...$key){
   foreach($key as $i=>$item){
    return function($a, $b) use($key, $i) {
      if($a[$key[$i]]>$b[$key[$i]]){
        return true;
      }
      if($a[$key[$i]] == $b[$key[$i]]) {
        if(isset($key[$i+1])){
          return $a[$key[$i+1]] < $b[$key[$i+1]];
        }
      }
    };
  }
}
$test_order_by_keys = $arr;
usort($test_order_by_keys, orderByKeys('age', 'id'));
var_dump($test_order_by_keys === $arr_sort_by_age_and_id);//true
```

### 内置函数 array_multisort 解决方案,更加强大和清晰
```php
  $arr_multisort = $arr;
  $sortedAge = array_column($arr_multisort, 'age');
  $sortedId  = array_column($arr_multisort, 'id');
  array_multisort(
    $sortedAge, SORT_ASC,
    $sortedId, SORT_DESC,
    // .... 更多字段的order by
    $arr_multisort);
  var_dump($arr_multisort === $arr_sort_by_age_and_id);//true
```

## Illuminate\Support\Collection包 Collection 集合对象的方法
### 简介
laravel框架中内置了Collection类,将Model调用的数据打包成一个对象,对象中集成了很多常用的处理数据的方法,如 every  集合对象真理测试,filter过滤,where查找,sort,group,json,contain,has等方法.
```php
    $user= User::where($conditions)->get();// 从数据库取出打包成Collection 对象
    // Eloquent where之后在Collection类中还可以再一次处理
    $result = $user->where($conditions_else)->when($when)->toJson();

```
### laravel是由一个一个包组合成的框架, 很多包依赖较低,可以在工程中直接引用
```
  composer require illuminate/support
  # support 包含了很多可复用的方法  Collection是其中之一 还有字符串和数组方法
```
#### 使用方法
1. 使用collection类
```php
  require 'vendor/autoload.php';
  use Illuminate\Support\Collection;
  $collect = new Collection($arr); // 获得一个collection对象
  $test_by_id = $collect->sortBy('id')->values()->all();
  // sortBy 底层采用asort排序,默认不会改变索引,values() 重置索引 ,all()取出数组
  $test_by_age = $collect->sortBy('age')->values()->all();
  var_dump($test_by_id===$arr_sort_by_id);
  var_dump($test_by_age===$arr_sort_by_age);
  #collection 没有支持orderBy 多个字段
```

2. 使用helpers.php里面的函数方法 (包含68个常用方法)

```
  # package.json
  ...
    "autoload":{
      "files":"vendor/illuminate/support/helpers.php"
    }
  ...

```

`composer dump-auto ` //重新生成autoload.php 文件

```php
require 'vendor/autoload.php';
//$test_by_id = array_sort($arr,'id');//array_sort 内部调用 collection 方法但是不改变索引
//$test_by_id = array_sort($arr,'age');
$collect = collect($arr);
//...
$test_by_id = $collect->sortBy('id')->values()->all();
$test_by_age = $collect->sortBy('age')->values()->all();
var_dump($test_by_id === $arr_sort_by_id);
var_dump($test_by_age === $arr_sort_by_age);

```

## 总结
- 可以在SQL中直接完成排序再取出数据使用,如果必要的话.
- 框架使用的是uasort/ usort /asort 实现排序
  - 如果没有使用框架,原生的uasort可以满足order by 1个字段,order by 多个字段需要测试检验
  - 使用库或者框架,建议使用 sortBy,名称具有语义,laravel获取的对象自带该方法,且经过了测试
  - 多个order 排序建议使用,array_multisort来进行,灵活而且强大,或者数据库排序
