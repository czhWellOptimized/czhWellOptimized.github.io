---
layout: post
title: Project3-query execution
tags: [database]


---

Project1 C++Primer

```c++
bool SeqScanExecutor::Next(Tuple *tuple, RID *rid) {
  TableIterator iter_ = *table_iter_;
  while (iter_ != table_heap_->End()) {
    Tuple tup = *iter_;
    iter_++;
    bool eval = true;
    if (plan_->GetPredicate() != nullptr) {
      eval = plan_->GetPredicate()->Evaluate(&tup, GetOutputSchema()).GetAs<bool>();
    }
    if (eval) {
      *tuple = tup;
      return true;
    }
  }
  return false;
}
```

当我写出如上所示代码以后，程序就陷入了死循环。单看代码当然是没问题的……因为这里我们需要推进迭代器，所以复制的话每次都是Begin迭代器。只有引用才可以持续推进迭代器。

```c++
TableIterator &iter_ = *(table_iter_.get());
```

