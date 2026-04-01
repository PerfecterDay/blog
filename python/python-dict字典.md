# python字典
{docsify-updated}




# dict字典
---------------
Python内置了字典：dict的支持，dict全称dictionary，在其他语言中也称为map，使用键-值（key-value）存储，具有极快的查找速度。dict是无序的。

## dict定义
直接用{key1:value1,key2:value2.....}的形式可以定义一个字典：

    d = {'Michael': 95, 'Bob': 75, 'Tracy': 85}
    d['Michael'] //95

## dict加入新键值对

    d['new']='new' //d={'Michael': 95, 'Bob': 75, 'Tracy': 85,'new':'new'}

## 修改某个key的value

    d['new']='old' //d={'Michael': 95, 'Bob': 75, 'Tracy': 85,'new':'old'}

## 判断key是否存在
通过in判断key是否存在：

    'new' in d //true
    'test' in d //false
访问不存在的key时，会报错。使用get()方法，可以返回None或指定值：

    d['test'] //KeyError
    d.get('test') //None
    d.get('test',-1) //-1

## 删除key
要删除一个key，用pop(key)方法，对应的value也会从dict中删除：

    d.pop('new') //删除key new及对应的值


