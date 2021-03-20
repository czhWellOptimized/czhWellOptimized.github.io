---
layout: post
title: Project1-BufferPool
tags: [database]
---

该Project下有4个主要的数据结构，分别是：

```c++
/** Array of buffer pool pages. */
Page *pages_;              
/** Page table for keeping track of buffer pool pages. */
std::unordered_map<page_id_t, frame_id_t> page_table_;
/** Replacer to find unpinned pages for replacement. */
Replacer *replacer_;
/** List of free pages. */
std::list<frame_id_t> free_list_;
```

从注释来看，`pages_` 包含了 `replacer_` `free_list_` 以及含有pin_count的所有frames。再根据page的三种状态之间的转换free、unpin、pin，我们可以进行编程。

在编程时关注 page的三个属性:`page_id`,`pin_count_`,`is_dirty_`,更新上述4个数据结构。

Bug1:C++的类内成员变量如果是内置类型，比如int，一定要进行初始化，否则你永远不知道编译器会给它设置成多少……

Bug2:`pages_`的索引在用page_id。。。大概我不是用脑子写的代码，是用脚写的。

Bug3:之前`lru_replacer`留下的蹩脚代码，之前的析构函数是这样处理的。当时想的太简单，测试用例又没检查出来。

```c++
    // head->prev=nullptr;
    // head->next=nullptr;
    // tail->prev=nullptr;
    // tail->next = nullptr;
```

其实应该这样处理：

```c++
    std::shared_ptr<Node> work=head;
    std::shared_ptr<Node> next=head->next;
    while(work){
        work->next=nullptr;
        work=next;
        if (next) next = next->next;
    }
```

Bug4：Single-parameter constructors should be marked explicit.

https://clang.llvm.org/extra/clang-tidy/checks/google-explicit-constructor.htm

这种类型的失误我目前没有太多时间去学习，需要注意。

Bug5:容器检查是否有元素就用empty()，别用size()

Bug6: auto 遍历容器的时候 用 `Range-based for loop`

Bug7：能用!表示逻辑就别==true,==false

