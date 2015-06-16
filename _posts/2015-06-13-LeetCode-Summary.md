---
layout: post
title:  "LeetCode十大难题"
description:
categories:
- Tech
tag: Tech, Interview
permalink: leetcode_summary
---

由于在white board上写算法成为了硅谷互联网公司筛选人才的主要手段, 各种面试素材近年来也是层出不穷. 本人暑假无事刚刚刷完LeetCode的OL, 在此有感而发...
<!--more-->	

面试一直是被人们津津乐道的话题. 除了算法和数据结构这些重中之重本木之源的基础知识, 面试题书籍也是准备CS面试时的杀手级资料. 从入门级的红书Programming Interview Exposed, 到分类详尽Cracking The Coding Interview. 这两本书可以提供一个对于面试很系统的分类和认识. 熟练掌握这些题目可以很好地面对大多数的面试. 除了这两本主流的书籍, LeetCode一直是面试准备不二素材之选. 除了良好的社区, 细致到tutorial, 还有一套完善的Online Judge系统可以像做Google Coding Jam一样提交代码进行测试. 我个人还是觉得LeetCode的平均难度是高于前面提到的两本书的, 其原因无外乎在于包含了大量的Tree, Graph相关的题目而这些一般都会存在相对复杂的抽象并且牵扯到recursion, 另一个就是LeetCode上存在一定数量的DP题目. DP在面试中出现的概率感觉是小之又小的, 一般只有在一些特别难进的大公司的最后一轮才有出现. 而其威力之可怕基本上是一题既出干倒一片. 熟练地掌握LeetCode上的DP类题目可以说是位面试DP类问题打下坚实的基础.  
截止到目前LeetCode的OL一共有226道题, 除了必须购买LeetCode EBook才能解锁的10道, 本人今天终于在无聊的假期里刷完了其余216道题. 心血来潮, 借此博客评选LeetCode十大OL难题以做完结纪念, 并且来这个博客除除草... 各位有兴趣也可以去挑战一下哈. 

1.	Wildcard Matching
__复杂DP__
2.	Scramble String
__复杂DP__
3. Best Time to Buy and Sell Stock IV
__复杂DP__
4. Palindrome Partitioning II
__DP__
5. Word Ladder II
__DP__
6. Interleaving String
__DP__
7. The Skyline Problem
__很有意思的题目, 需要高效的转化来实现效率算法__
8. Maximum Gap
__利用数学原理提速, 以通过大的test__
9. Word Search II 
__一个高效的数据结构的重要性__
10. Reverse Nodes in k-Group
__胆大心细. 问题一看就懂, code一交就错__


先写这么多了, 有空出个系列详解上面这十道题...最后吐槽一下LeetCode的限时机制. OL是有时间限制的, 而超时就会fail. 不过他的时间限制似乎是绝对时间而不是相对时间, 所以一些不公平. 同样的Code用java写的死活都是超时, 把同样的写成C就能秒秒通过. 比如Count Complete Tree Nodes, 有个巨大的Test Case, 用JAVA写的一直是超时过不了, 当我用C重写以后, 56ms就pass了. Java哭晕在厕所.
