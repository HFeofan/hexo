---
title: Shell编程——数组
date: 2017-03-29 22:02:13
tags: Shell
---

作为一个程序员，不会写Shell怎么行呢。

# 初始化

```bash
fruit_arr=(apple orange watermelon ananas banana)
```
对于shell程序中的空格使用很默认，写惯了C语言，按照习惯等号两边需要配备空格。但是在shell中这个写法会出错。所以一开始有些不习惯。不过可以这样理解，shell程序就是一个文本，在执行的时候需要先解析文本，如果允许空格存在，在词法分析会比较麻烦，所以空格在一般的程序语句中是不必要的。那么什么时候需要空格呢。觉得像数字运算或者表达式中，例子如下：

```bash
if [ $UID -ne 0 ]; then
echo Non root user
else
echo Root user
fi
```
如判断使用的`[]`里面，或者`(())`这样的括号里面的表达式。

以上是一个初学者对shell中空格使用的浅显的认识。

# 操作

```bash
#添加元素
fruit_arr=($fruit_arr[*] strawberry)

#删除元素
unset fruit_arr[1]

#分片操作
${fruit_arr[*]:1:2}

#模式替换 ${数组[@/*]/模式/新值}
${fruit_arr[*]/orange/strawberry}
```
在shell中好像除了声明变量不需要使用`$`，其他地方使用到的时候都需要使用`$`，但是`unset fruit_arr[1]`却不需要。这点感觉很奇怪。对shell的规则还需要继续探索。

`unset`操作只是删除了对应位置上的值，并没有将该位置一并移除，也就是说`unset`并不会改变数组长度。

练习代码
```bash
fruit_arr=(apple orange watermelon)
unset fruit_arr[1]
echo $fruit_arr 
#orange watermelon
echo ${#fruit_arr[@]}  
#3
result=${fruit_arr[@]:1:3}
echo ${#result[*]}
#orange watermelon
echo ${#result[*]}
#17
result2=($result[@] apple)
echo $result2
#orange watermelon apple
echo ${#result2[@]}
#2
echo $result2[1]   
#orange watermelon
echo $result2[2]
#apple
```
可以看到，切片的结果会当成一个字符串的形式，shell又一个令人迷惑的地方。以后细究的时候在补一下吧。

# 应用

输出单个元素
```bash
echo $fruit_arr[1]
```
shell中数组的下标是从1开始的，`echo $fruit_arr[0]`的结果为空。

```bash
#for循环输出数组中的元素
for fruit in $fruit_arr
do
echo $fruit
done

#数组长度
echo ${#fruit_arr[@]}
echo ${#fruit_arr[*]}
```

