---
title: Haskell常用扩展
date: 2018-11-20 13:51:09
tags:
     - haskell 
---

### BinaryLiterals

### ExistentialQuantification

### FlexibleInstances

### LambdaCase
一个句法扩展，允许你用\case代替\arg - > case arg of.
请考虑以下函数定义：
```
dayOfTheWeek :: Int -> String
dayOfTheWeek 0 = "Sunday"
dayOfTheWeek 1 = "Monday"
dayOfTheWeek 2 = "Tuesday"
dayOfTheWeek 3 = "Wednesday"
dayOfTheWeek 4 = "Thursday"
dayOfTheWeek 5 = "Friday"
dayOfTheWeek 6 = "Saturday"
```
如果您想避免重复函数名称，可以这样编写：
```
dayOfTheWeek :: Int -> String
dayOfTheWeek i = case i of
    0 -> "Sunday"
    1 -> "Monday"
    2 -> "Tuesday"
    3 -> "Wednesday"
    4 -> "Thursday"
    5 -> "Friday"
    6 -> "Saturday"
```
而使用LambdaCase扩展后，您可以将其编写为函数表达式，而无需为参数命名：
```
{-# LANGUAGE LambdaCase #-}

dayOfTheWeek :: Int -> String
dayOfTheWeek = \case
    0 -> "Sunday"
    1 -> "Monday"
    2 -> "Tuesday"
    3 -> "Wednesday"
    4 -> "Thursday"
    5 -> "Friday"
    6 -> "Saturday"
```

### ScopedTypeVariables
ScopedTypeVariables允许您引用声明中的通用量化类型, 更明确一点：
```
import Data.Monoid

foo :: forall a b c. (Monoid b, Monoid c) => (a, b, c) -> (b, c) -> (a, b, c)
foo (a, b, c) (b', c') = (a :: a, b'', c'')
    where (b'', c'') = (b <> b', c <> c') :: (b, c)
```
重要的是我们可以使用a,b和c来指示编译器在声明的子表达式中(where子句中的元组和最终结果中的第一个a),实际上,ScopedTypeVariables有助于将复杂函数编写为部分之和，允许程序员将类型签名添加到没有具体类型的中间值。


### OverloadedStrings
通常，Haskell中的字符串文字具有String类型（它是[Char]的类型别名）。虽然这对于较小的教育程序来说不是问题，但实际应用程序通常需要更高效的存储，例如Text或ByteString.
OverloadedStrings只是将文字类型更改为:
```
"test" :: Data.String.IsString a => a
```
允许它们直接传递给期望这种类型的函数。许多库为类似字符串的类型实现了这个接口，包括Data.Text和Data.ByteString ，它们都比[Char]提供了一定的时间和空间优势。

还有一些像Postgresql简单库那样的OverloadedStrings独特用途，它允许用双引号编写SQL查询，就像普通字符串一样，但提供了对不正确连接的保护，这是一种臭名昭着的SQL注入攻击源。
要创建IsString类的实例，您需要实现fromString函数。示例：
```
data Foo = A | B | Other String deriving Show

instance IsString Foo where
  fromString "A" = A
  fromString "B" = B
  fromString xs  = Other xs

tests :: [ Foo ]
tests = [ "A", "B", "Testing" ]
```



### TupleSections
一种语法扩展，允许以节的方式应用元组构造函数(它是一个运算符):
```
(a,b) == (,) a b

-- With TupleSections
(a,b) == (,) a b == (a,) b == (,b) a
```
#### N-tuples
它也适用于元素大于2的元组
```
(,2,) 1 3 == (1,2,3)
```
#### Mapping
这在使用部分的其他地方也很有用:
```
map (,"tag") [1,2,3] == [(1,"tag"), (2, "tag"), (3, "tag")]
```
没有此扩展的上述示例如下所示：
```
map (\a -> (a, "tag")) [1,2,3]
```



