---
date: "2017-04-11T08:43:29+08:00"
title: "函数式编程笔记(2)-Hindley-Milner类型签名"
description: Hindley Minlner signature
---

刚接触函数式编程的人总是搞不明白函数类型签名，我记得去年同事有分享过一次 FP 的 session，用 Hashkell 来做题，其中的类型签名当时就不太搞明白。类型是让不同背景的人高效沟通的元语言，大部分都是以一种叫`Hindley Milner`系统来表述的。

类型签名的主要作用:

- 标明函数输入和输出
- 让函数保持通用，抽象
- 可以用于编译时候检查
- 是代码最好的文档

# Hindley-Milner story

其实 Hindley Miler 并不是一个很复杂的系统，不过我们需要一些学习和练习过程来熟悉和掌握它。

```javascript
//  capitalize :: String -> String
var capitalize = s => {
  return toUpperCase(head(s)) + toLowerCase(tail(s))
}

//=> "Smurf"
capitalize("smurf")
```

这里定义了一个函数 capitalize，它接收一个 string，然后返回一个 string，在 HM 系统中，函数都写成 a -> b 这样，所以这个函数的签名`capitalize :: String -> String`可以理解为一个接受 string 并返回 string 的函数。

再来一些例子：

```javascript
//  len :: String -> Number
var len = function(s) {
  return s.length
}

//  join :: String -> [String] -> String
var join = curry(function(delimiter, data) {
  return data.join(delimiter)
})

//  match :: Regex -> String -> [String]
var match = curry(function(reg, s) {
  return s.match(reg)
})

//  replace :: Regex -> String -> String -> String
var replace = curry(function(reg, sub, s) {
  return s.replace(reg, sub)
})
```

对于稍微复杂的函数类型签名，我们开始可以这么理解，把最后一个类型看作是它的返回值，其他的都看作是参数，那么 join 就是接受一个 string 和 array，然后返回一个 string，match 就是接受一个正则和 string，返回一个 array，replace 就是接受一个正则，二个 string，返回一个 string，这样看起来就比较简单啦。

不过这只是帮助我们理解函数签名的开始，下面我们再看一个有意思的理解：

对于 join，我们把它的签名分组一下，`join :: String -> ([String] -> String)`，join 接受一个 string 参数，返回一个接受 array，返回 string 的函数，这样会更容易帮助我们深入理解柯里化的作用。

对于 replace，分组一下类型签名， `replace :: Regex -> (String -> (String -> String))`, 它接受一个正则，返回一个接受一个 string，再返回一个接受 string，返回 string 的新函数，这个新函数调用后，又返回一个接受一个 string，最后返回一个 string 的函数，最后一个函数调用后返回最终结果。

因为柯里化就是这样，如果一个函数接受多个参数，那么当它被柯里化之后，如果你调用时候只传入一个参数，那么它将返回一个新的接受剩余参数的柯里化函数，这样从函数签名角度来看，当你每传入一个参数，就弹出类型签名最前面的那个类型，这样就很容易理解了。

当然你也可以一次性传入全部参数，那么将返回最终结果。

在 HM 系统中，一般约定俗成用 a, b 类似这样的来代表类型，a 引用代表的一定是同一个类型，a 和 b 一定是不同的类型，比如

- `len :: a -> a`
- `join :: a -> [a] -> a`
- `match :: a -> b -> [b]`
- `replace :: a -> b -> b -> b`

类型签名能够一字一句地告诉我们函数做了什么事情，读懂类型签名是在函数式编程过程中一项非常重要的技能，会让你受益无穷。

再来两个练习试试

```javascript
//  filter :: (a -> Bool) -> [a] -> [a]
var filter = curry(function(f, xs) {
  return xs.filter(f)
})

//  reduce :: (b -> a -> b) -> b -> [a] -> b
var reduce = curry(function(f, x, xs) {
  return xs.reduce(f, x)
})
```

# Parametricity

在上面看到的类型签名中，有各种不同的类型，这就会引入一个程序语言特性叫[parametricity](http://en.wikipedia.org/wiki/Parametricity)。这个特性表明函数将会以一种统一的行为作用于所有的类型。

看一个这样的类型签名： `head :: [a] -> a`， 从类型签名上我们可以得到这个函数它接受一个元素是类型 a 的数组，返回一个类型 a 的元素。类型 a 可以是任意类型，比如 Number，String，Boolean 等，那么这个函数对不同的任意类型的作用都是保持统一的，这就是 parametricity 的含义。这个函数可能返回数组的第一个元素，可能是最后一个，也可能是随机的一个，这时候函数的名字会有更多的信息给我们，不管是哪种情况，在这里类型 a 的多态性会大幅度缩小函数可能性的范围，保证在多态性的情况下，函数的作用都是统一的。

# Free theorems

看个简单例子

```javascript
// head :: [a] -> a
compose(f, head) == compose(head, map(f))
```

在这个例子中，左边的操作是，先对数组 a 进行 head 操作，得到一个 a，然后再对它调用 f 函数得到结果。右边的操作是，先对数组 a 所有元素都调用 f 函数得到一个新数组，然后再对新数组做 head 操作得到结果。这两个操作最终结果是相等的，只不过左边的效率要高一些。

这种被称为自由定理，可以应用到多态类型的签名上。

# Type constrains

类型签名可以把类型限制在一个特定的接口上，

```javascript
reverse :: Ord a => [a] -> [a]
```

在这里`Ord a`约束了 a 必须是一个 Ord 对象，或者 a 必须实现 Ord 接口，这样就限制了函数的作用范围，这样的声明被称作类型约束。

#### 参考

https://drboolean.gitbooks.io/mostly-adequate-guide/content/ch7.html
