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

- Bug1:C++的类内成员变量如果是内置类型，比如int，一定要进行初始化，否则你永远不知道编译器会给它设置成多少……


- Bug2:`pages_`的索引在用page_id。。。大概我不是用脑子写的代码，是用脚写的。


- Bug3:之前`lru_replacer`留下的蹩脚代码，之前的析构函数是这样处理的。当时想的太简单，测试用例又没检查出来。


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

- Bug4：Single-parameter constructors should be marked explicit.


https://clang.llvm.org/extra/clang-tidy/checks/google-explicit-constructor.htm 这种类型的失误我目前没有太多时间去学习，需要注意。

- Feature?：如果Unpin的时候Frame已经在replacer中，不必处理，就当没发生事情一样，否则就会过不了测试。


-------------------------------------------------------------------------------

## page_table_ 中是否应该只存放pinned的Frame？

`pages_``page_table_``replacer_``free_list_`

从存放的Frame来看，`pages_`是全集，`free_list_`是删除了Page的Frame，`replacer_`是page.pin_count==0的Frame。

但是剩下的`page_table_`就有点疑问了：它到底是存着pinned page的Frame，还是全部Frame...在本实验中我是把它当成全集来看的。当然可能只放pinned page的Frame会好一点，毕竟这样就达成了一种互斥的感觉，很完美。那么如何实现这种方法呢？

首先明确一点：`free_list_`中的Frame不需要查找，直接取出来即可。然后考虑`Page *BufferPoolManager::FetchPageImpl(page_id_t page_id)`，给定page_id取出该Page，现在Page有可能是pin_count==0，那么一定不在`page_table_`中了，我们不能直接通过`page_table`的哈希表查找。必须在`replacer_`中增加一个查找接口，这个也好做，因为`replacer_`中本身也有一个哈希表，此处就不实现了。