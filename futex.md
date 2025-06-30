# TatlinOS的futex

TatlinOS的futex机制相比于TrustOS有较大改动。

- 我们支持了含bitset的Wait和Wake操作，这在TrustOS中是没有的。
- 我们把无bitset的Wait和Wake操作视为bitset=全1的Wait和Wake操作。至少，只有这样可以过测试集。
- 我们把对含timeout的futex会引起的bug做了修复，见下方Futex版本号机制。

## Futex版本号机制

假设现在有这样的情况：

- （t=0.0）线程A：一秒后如果没人叫醒我，就叫醒我
- 内核：设置一个一秒后的计时器，到时间就把线程A唤醒
- （t=0.3）线程B把线程A唤醒
- （t=0.4）线程A又开启了一次futex等待，这次没有截止时间
- （t=1.0）计时器到时，A被唤醒

这样，A就被错误地唤醒了，唤醒它的计时器不是它本次Wait设置的！

为了解决这个问题，我们可以给每一次Wait添加一个版本号，在计时器的队列的结构体中，也添加这样一个版本号，这样，当计时器到时并调用futex模块的函数时，就可以把版本号一块传过来，futex模块就可以通过比对版本号，避免错误地把线程唤醒。

Linux似乎也使用了类似的设计。

## 关于FUTEX_PRIVATE_FLAG

TatlinOS目前**完全不考虑FUTEX_PRIVATE_FLAG**，因为这只是个性能优化位，而不是一个语义检查位，内核没有检查“这个Futex到底是不是只被你这个进程使用”的职责，因此内核完全可以选择性无视这一位。

## 关于RobustList

TatlinOS完全没有实现RobustList，当线程退出时，TatlinOS从来不会检查用户态空间中的链表。