---
title: Haskell-IO
date: 2019-05-29 15:18:26
tags:
    - haskell
    - IO
---
>>> 本文翻译自维基百科(https://wiki.haskell.org/IO_inside),想要看原文的可以去那里查看.

### IO内部
Haskell I/O一直是新Haskellers混乱和惊喜的根源。虽然Haskell中的简单I/O代码看起来非常类似于命令式语言中的等价物，但尝试编写更复杂的代码通常会导致完全混乱。这是因为Haskell I/O内部真的非常不同。Haskell是一种纯语言，甚至I/O系统也无法打破这种纯度。
以下正文试图解释Haskell I/O实现的细节，这个解释应该可以帮助你最终掌握所有的智能I/O技巧。此外，我已经添加了您可能遇到的各种陷阱的详细说明。阅读本文后，您将获得“Haskell I/O大师”学位，该学位同等于计算机科学和数学学士学位。
如果您是Haskell I/O新手，您可能更愿意从阅读IO简介开始(https://wiki.haskell.org/Introduction_to_IO)

#### Haskell是一门纯语言
Haskell是一门纯语言，这意味着任何函数调用的结果完全由其参数决定，像C中的rand()或getchar()这样的伪函数在每次调用时都会返回不同的结果，这些函数根本不可能在Haskell中编写。而且，Haskell函数不能有副作用，这意味着这些函数不能对"真实世界"做任何更改，例如：更改文件/写入文件/打印/通过网络发送数据等。这两个限制一起意味着任何函数调用都可以被具有相同参数的先前调用的结果替换，并且语言保证所有这些重新排列不会改变程序结果!

让我们将其与C语言进行比较: 优化C编译器尝试猜测哪些函数没有副作用，并且不依赖于可变全局变量。如果这个猜测错了，优化可以改变程序的语义！为了避免这种灾难，C优化器在猜测中是保守的，或者需要程序员提供有关函数纯度的提示。

与优化的C编译器相比，Haskell编译器是一组纯数学转换。这导致更好的高级优化设施。此外，纯数学计算可以更容易地分成几个可以并行执行的线程，这在多核CPU的这些日子里越来越重要。最后，纯计算不易出错且更容易验证，这增加了Haskell的稳健性和使用Haskell的程序开发速度。Haskell纯度允许编译器只调用其结果确实需要计算高级函数的最终值的函数(例如: main) - 这称为惰性求值。纯粹的数学计算是件好事，但是I/O动作怎么样？函数如下:
```
putStrLn "Press any key to begin formatting"
```
不能返回任何有意义的结果值，那么我们如何确保编译器不会省略或重新排序其执行？总的来说，我们如何使用完全惰性的语言处理有状态的算法和副作用？这个问题在18年的Haskell开发中提出了许多不同的解决方案(参见Haskell的历史)，尽管现在基于monads的解决方案已成为标准。

#### monad是什么?
什么是monad? 这是来自数学范畴理论的东西，我不知道了。为了理解monad如何用于解决I/O和副作用的问题，您不需要知道它。就像我一样，只知道小学数学就足够了。
让我们想象一下，我们想在Haskell中实现众所周知的'getchar'函数。它应该有什么样的类型？我们试试吧：
```
getchar :: Char

get2chars = [getchar,getchar]
```
只有'Char'类型的'getchar'函数会得到什么？您可以在'get2chars'的定义中看到所有可能出现的问题：
* 因为Haskell编译器将所有函数视为纯函数(没有副作用)，所以它可以避免对"getchar"的"过度"调用, 并使用两次返回值.
* 即使它确实进行了两次调用，也无法确定应首先执行哪个调用。你想按照阅读顺序或相反的顺序返回两个字符吗？"get2chars"定义中没有任何内容可以回答这个问题。

从程序员的角度来看，如何解决这些问题呢？
让我们为"getchar"函数增加一个"伪装"的参数，使每个调用与编译器的观点"不同":
```
getchar :: Int -> Char

get2chars = [getchar 1, getchar 2]
```
马上，这解决了上面提到的第一个问题 - 现在编译器将进行两次调用，因为编译器将它们视为具有不同的参数。整个'get2chars'函数也应该有一个"伪装"参数，否则我们会遇到同样的问题：
```
getchar   :: Int -> Char
get2chars :: Int -> String

get2chars _ = [getchar 1, getchar 2]
```
现在我们需要给编译器一些线索来确定它应该首先调用哪个函数。Haskell语言没有提供任何表达评估顺序的方法......除了数据依赖性！如何添加一个人工数据依赖项，以防止在第一个'getchar'之前评估第二个'getchar'？为了实现这一目标，我们将从'getchar'函数返回一个额外的"伪装"结果，该"伪装"结果将用作下一个'getchar'函数调用的参数：
```
getchar :: Int -> (Char, Int)

get2chars _ = [a,b]  where (a,i) = getchar 1
                           (b,_) = getchar i
```
到目前为止还不错 - 现在我们可以保证在读'b'之前读取'a'，因为读'b'需要通过读'a'返回的值('i')!
我们在'get2chars'中添加了一个"伪装"参数，但问题是Haskell编译器太聪明了! 它可以相信外部'getchar'函数真的依赖于它的参数，但是对于'get2chars'函数，它会看到我们是在作弊，因为我们扔掉了它(参数使用了_占位符)! 因此，编译器不会觉得有必要按照我们想要的顺序执行调用。我们该如何解决这个问题? 将这个"虚假"的参数传递给'getchar'函数怎么样?! 这样的话，编译器就无法猜测它是否真的未使用过。
```
get2chars i0 = [a,b]  where (a,i1) = getchar i0
                            (b,i2) = getchar i1
```
还有更多 - 'get2chars'具有与'getchar'功能相同的纯度问题。
如果需要调用'get2chars'两次，则需要一种方法来描述这些调用的顺序。看着：
```
get4chars = [get2chars 1, get2chars 2]  -- order of 'get2chars' calls isn't defined 未定义'get2chars'调用的顺序
```
我们已经知道如何处理这些问题 - 'get2chars'函数也应该返回一些可以用来顺序调用的"伪装"值：
```
get2chars :: Int -> (String, Int)

get4chars i0 = (a++b)  where (a,i1) = get2chars i0
                             (b,i2) = get2chars i1
```
但是'get2chars'函数应该返回什么"伪装"值？如果我们使用一些整数常量，那么过于聪明的Haskell编译器会猜测我们想再次作弊。如何返回'getchar'函数返回的值？看：
```
get2chars :: Int -> (String, Int)
get2chars i0 = ([a,b], i2)  where (a,i1) = getchar i0
                                  (b,i2) = getchar i1
```
信不信由你，但我们刚刚已经构建了整个"monadic" Haskell I/O系统。

#### 欢迎来到真实世界，宝贝
警告：关于IO的以下故事是不正确的，因为它无法实际解释IO的一些重要方面（包括交互和并发）。但是，有些人发现开始理解是有用的。
Haskell 'main'函数具有以下类型:
```
main :: RealWorld -> ((), RealWorld)
```
其中'RealWorld'是一种"伪装"的类型，用来替换我们的Int。
这就像在接力赛中接过的接力棒。当'main'函数调用某些IO函数时，它将作为参数收到的"RealWorld"传递给了IO函数。所有IO函数都有类似的类型，涉及RealWorld作为参数和结果。确切地说，"IO"是以下列方式定义的类型同义词：
```
type IO a  =  RealWorld -> (a, RealWorld)
```
因此，'main'函数只有类型"IO()"，'getChar'函数有类型"IO Char",等等。您可以将"IO Char"类型视为"获取当前的RealWorld，对其执行某些操作，并返回Char和（可能已更改的）RealWorld"。让我们看'main'调用'getChar'两次：
```
getChar :: RealWorld -> (Char, RealWorld)

main :: RealWorld -> ((), RealWorld)
main world0 = let (a, world1) = getChar world0
                  (b, world2) = getChar world1
              in ((), world2)
```
仔细看看：'main'函数将收到的"world0"传递给第一个'getChar'函数。 这个'getChar'函数返回一些RealWorld类型的新值，它将在下一次调用中使用。最后，'main'返回它从第二个'getChar'获得的"world2"。
* 如果没有使用它读取的字符，这里是否可以省略任何'getChar'调用？不，因为我们需要返回第二个'getChar'结果的"world2"，这又需要从第一个'getChar'返回的"world1"。
* 是否可以重新排序'getChar'调用？ 否：第二个'getChar'在第一个之前无法调用，因为它使用了第一次调用返回的"world1".
* 是否可以重复调用？ 在Haskell语义中 - 是的，但真正的编译器永远不会在这种简单的情况下重复工作（否则，生成的程序将没有任何速度保证）。

正如我们已经说过的那样，RealWorld值被用作一个接力棒，它在严格的顺序中被'main'调用的所有例程之间传递。在每个例程中，RealWorld值以相同的方式使用。 总的来说，为了“计算”从'main'返回的世界，我们应该直接或间接地执行从'main'调用的每个IO过程。这意味着插入链中的每个程序都将在我们打算调用它时执行(相对于其他IO操作)。让我们考虑以下程序：
```
main = do a <- ask "What is your name?"
          b <- ask "How old are you?"
          return ()

ask s = do putStr s
           readLn
```
现在你有足够的知识以低级别方式重写它，并检查每个应该执行的操作是否真的将使用它应该具有的参数并按照我们期望的顺序执行。
但是条件执行呢？没问题。让我们定义众所周知的"when"操作：
```
when :: Bool -> IO () -> IO ()
when condition action world =
    if condition
      then action world
      else ((), world)
```
如您所见，我们可以根据数据值轻松地在执行链中包含或排除IO过程(操作)，如果在'when'的调用中'condition'将为False，那么'action'将永远不会被调用，因为真实的Haskell编译器，从不调用其结果不用于最终结果的函数(例如: 这里main函数的"world"最终值)。循环和更复杂的控制结构可以以相同的方式实现。试试看吧！
最后，你可能想要知道，在程序中传递这些RealWorld值需要花费的成本，它是免费的，这些"伪装"值仅在编译器分析和优化代码时存在，但是当它进入汇编代码生成时，它"突然"意识到这种类型就像"()"，因此所有的这些参数和结果值都可以从最终生成的代码中省略。这不漂亮吗？

#### '>>=' 和 'do' 表示法

#### 可变数据(引用/数组/哈希表)

#### IO动作作为值

#### 异常处理


#### 与C/C++和外部库的接口

#### IO monad的黑暗面

#### 更安全的方法: ST monad

#### 欢迎来到机器: 现实的GHC实施

#### 进一步阅读

#### to-do列表

















