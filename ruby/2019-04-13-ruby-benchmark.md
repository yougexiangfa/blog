---
title: Ruby中的Benchmark
layout: post
category: ruby
---

Benchmark 是用来衡量代码执行时间的一个工具，是程序优化的基本工具。

## Process 相关知识
在使用 Benchmark 之前，我们需要先了解几个概念。

* 进程(Process)

我们的代码是在操作系统的进程里运行的，程序执行的时间信息，也是由进程给出来的。

```ruby
# 查看当前进程ID，等同于全局变量 $$
Process.pid

# 查看当前进程的父进程，如果我们通过终端运行，父进程一般是 shell 进程
Process.ppid

# 查看当前的子进程
Process.waitall

Process.clock_gettime(Process::CLOCK_REALTIME)  # 真实时间
Process.clock_gettime(Process::CLOCK_MONOTONIC)  # 系统启动后的流逝时间；
```

* 进程时间信息(Process.times)

```ruby
Process.times
```

会返回一个 Process::Tms 实例，包含如下四个信息：
1. stime: 系统CPU时间(System CPU Time)
2. utime: 用户CPU时间(User CPU Time)
3. cstime: Children stime
4. cutime: Children utime

这里我不是很明白的是，此时进程还没有fork，没有子进程的情况下，Children CPU Time 是什么意思？

* `系统CPU时间`和`用户CPU时间`的解释
  * 系统CPU时间
  进程运行时，在系统区执行的时间，如（write，read等系统调用），运行的地方位于系统内存中。
  * 用户CPU时间
  进程运行时，在用户区执行的时间。这里主要是我们自己编写的代码，运行在用户内存中。程序从用户态到系统态需要消耗一定的时间，频繁的切换会导致系统运行的效率低下。



## Benchmark 实践
在了解了这几个概念之后，我们就可以读懂 Benchmark 返回的结果中的时间概念了。

* measure

```ruby
Benchmark.measure { 'x' * 10_0000_0000 }
```

会返回一个 Benchmark::Tms 实例，与 Process::Tms 一致，包含如下四个信息：
1. stime
2. utime
3. cstime
4. cutime
5. total: total 等于 stime, utime, cstime, cutime 四者之和。

* benchmark / bm

benchmark 方法会生成 Benchmark::Report 实例在代码块中执行。
bm 是 benchmark 方法的简单版本，使用了 Benchmark::Tms 中的 CAPTION 和 FORMAT 默认配置。

```ruby
n = 2_000_0000
Benchmark.bm do |x|
  x.report { n.times { 'x' } }
  x.report { n.times { "x" } }
end

# =>
#       user     system      total        real
#   1.786338   0.037146   3.515864 (  1.833894)
#   1.850868   0.038122   3.200687 (  1.907266)
```

上面这个例子对比了单引号`'`和双引号`"`的性能，我们应该主要关注 User CPU Time 和 Real Time, 这两个值更能体现真实的代码运行时间。
可以看出单引号的性能是优于双引号的。