---
title: 深入浅出Rust Future - Part 1
date: 2018-12-02 16:02:32
tags: Rust Future
---

# 前言

If you are a programmer and you like Rust you might be aware of the future movement that is buzzing around the Rust community. Great crates have embraced it fully (for example Hyper) so we must be able to use it. But if you are an average programmer like me you may have an hard time understanding them. Sure there is the official Crichton's tutorial but, while being very thorough, I've found it very hard to put in practice.

I'm guessing I'm not the only one so I will share my findings hoping they will help you to become more proficient with the topic.

如果你是一个程序员并且也喜欢Rust这门语言, 那么你应该经常在社区听到讨论`Future` 这个库的声音, 一些很优秀的`Rust Crates`都使用了`Future` 所以我们也应该对它有足够的了解并且使用它. 但是大多数程序员很难理解`Future`到底是怎么工作的, 当然有官方 `Crichton's tutorial`这样的教程, 虽然很完善, 但我还是很难理解并把它付诸实践.

我猜测我并不是唯一一个遇到这样问题的程序员, 所以我将分享我自己的最佳实践, 希望这他们能帮助你理解这个话题.

## 一段话概括`Future`

Futures are peculiar functions that do not execute immediately. They will execute in the future (hence the name). There are many reasons why we might want to use a future instead of a standard function: performance, elegance, composability and so on. The downside is they are hard to code. Very hard. How can you understand causality when you don't know when a function is going to be executed?

`Future` 是一个不会立即执行的 奇怪的`functions`. 他会

For that reason programming languages have always tried to help us poor, lost programmers with targeted features.

