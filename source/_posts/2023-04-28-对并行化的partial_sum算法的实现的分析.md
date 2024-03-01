---
layout: post
title: 对并行化的partial_sum算法的实现的分析
category: 并行
tags: C++ 并行
---

并行化的 partial_sum 算法的一种实现如下：

```C++
template<typename Iterator>
void parallel_partial_sum_2(Iterator first, Iterator last) {
    using value_type = typename Iterator::value_type;

    struct process_element {
        void operator()(Iterator first, Iterator last, std::vector<value_type> &buffer, unsigned i, barrier &b) {
            value_type &ith_element = *(first + i);
            bool update_source = false;

            for (unsigned step = 0, stride = 1; stride <= i; ++step, stride *= 2) {
                const value_type &source = (step % 2) ? buffer[i] : ith_element;
                value_type &dest = (step % 2) ? ith_element : buffer[i];

                value_type const &addend = (step % 2) ? buffer[i - stride] : *(first + i - stride);
                dest = source + addend;
                update_source = !(step % 2);
                b.wait();
            }


            if (update_source) {
                ith_element = buffer[i];
            }

            b.done_waiting();
        }
    };

    unsigned long const length = std::distance(first, last);
    if (length <= 1) {
        return;
    }
    std::vector<value_type> buffer(length);

    barrier b(length);

    std::vector<std::thread> threads(length - 1);
    join_threads joiner(threads);

    Iterator block_start = first;
    for (unsigned long i = 0; i < (length - 1); ++i) {
        threads[i] = std::thread{process_element(), first, last, std::ref(buffer), i, std::ref(b)};
    }

    process_element()(first, last, buffer, length - 1, b);
}
```

Q：barrier 的作用是什么？

S：在这个算法的实现中，每轮循环需要将间隔（stride\*=2）-1 的元素加到这个线程负责计算的元素上（这个元素由参数 i）指出，在每一轮循环中所有的线程都必须同时工作，将它负责的元素与前面间隔若干个元素的元素相加，因为某个线程计算的结果可能在之后被别的线程使用，所以算法的实现必须保证循环的运行是同步的，即：**所有线程都必须同时处于相同的第 m 次迭代中，以保证不会出现”线程 A 要使用线程 B 的结果但线程 B 的结果可能还没有计算出来“这种情况**

在主线程中，barrier 的阻塞数被设置为要求前缀和的元素的总数，与开启的线程数相同，每有一个线程到达一轮循环的结尾处就会被阻塞，直到所有线程上的同一轮循环完成；

Q：done_waiting 方法的作用是什么？

S：在序列中，不同的元素求前缀和所需要的循环轮数是不同的，对于第 0 个元素来说，它在循环开始之前就计算完成了，甚至不需要进循环（因为它是第一个元素，根本没有前缀）；对于第 1 个元素来说，只循环一轮（将它和它的前一个元素相加就够了）；但是 barrier 的要求是让所有线程每次循环都调用 wait 的，这就导致了有的线程明明可以在结束工作之后退出（因为它们本身并不需要工作那么多轮循环），但是还要继续循环调用 wait 以使得其它还在工作中的线程能够正常解除阻塞；这会导致有些线程明明没有工作要做还要在循环中空转（白白付出线程调度的开销）

解决的方法就是每有一个元素求前缀和完成后，就将屏障的阻塞数减一，那么即使这个线程退出了（意味着它没法调用 wait），余下的线程还是可以足量的调用 wait 以使所有线程都成功同步时解除阻塞；

Q：设置缓冲数组的原因是什么？为什么不能在原始序列中更新值？

S：主要是为了避免读写竞争，要知道在多个线程中，每一个元素的值既要被读取（用来和后面第 n 元素相加），还要被写入（用它和它前面第 n 个值相加并更新它自己），这就不能保证读写一致；

使用了缓冲区后，在所有线程的第一轮循环中，所有线程都从原始序列中读取值，将计算后的结果写入缓冲区，原始序列和缓冲区均不存在多个线程并行读写的问题；在下一轮循环两者反转，所有线程从缓冲区读取值，将读取的值写入原始序列，也不存在读写竞争的问题；

Q：可以注意到如果一个线程在缓冲区中存放了目标元素的值，那他就要在完成计算后将将值同步到原始数组，这个过程可能与其它线程一起读写原始序列的同一元素，这构成竞争吗？

S：其实不会，原序列和缓冲区交替地作为源和目标就决定了这个问题不会发生，线程地读写无非两种情况：要么从原序列中读，向缓冲区中写，要么从缓冲区中读，向原序列中写；

如果将值从缓冲区同步到原序列这个行为有效，就意味着上一轮循环中的计算是从原序列中取值，并将计算结果写入缓冲区的，这就意味着下一轮循环一定是从缓冲区中取值并写入原序列的，这时已经完成计算的线程也从缓冲区取值并写入原序列，所以读写任然是一致的，不会发生竞争问题；

Q：上一个 Q-S 中只讨论到了相邻的两次循环的情况，如果完成计算的线程在从缓冲区中同步值到原序列的同时其它线程已经进行了多轮循环，那么竞争会发生吗？

S：不会，实际上**完成计算的线程在从缓冲区中同步值到原序列的同时其它线程已经进行了多轮循环**这个情况就不可能出现，注意`process_element::operator()`中的 done_waiting 实在同步值操作的前面的（21~25 行），done_waiting 实际上是个释放操作，在其它线程进入下轮循环后并调用了 wait 后会一直阻塞（因为有个线程的工作已经完成了，不会再调用 wait 了），直到这个完成工作的线程调用 done_waiting，所以 done_waiting 是其它线程下一个循环完成的前序操作，而同步值操作是 done_waiting 的前序操作，所以同步值操作总是在其它线程下一轮循环完成之前完成，事实上是通过 barrier 的阻塞操作实施了同步；
