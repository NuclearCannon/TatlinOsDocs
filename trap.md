# TatlinOS的Trap

在TrustOS中，当内核`__return_to_user`时，会把当前内核控制流保存在TrapContext之中，这个控制流信息会在`__trap_from_user`中被恢复。也就是说，它的`__return_to_user`是可以返回的，会在用户态下一次中断时返回，紧接着`trap_return`返回，控制流进入`trap_loop`，再进入`trap_entry`……

但是，考虑到每一次返回的目的位置是同样的（`__return_to_user`的出口处），其实没有必要做这样的保存。

在TatlinOS中，没有`trap_loop`循环，用户态中断进入`__trap_from_user`后，不恢复现场，而是进入`trap_entry`函数：

```rust
#[no_mangle]
pub extern "C" fn trap_entry() {
    debug!("trap_entry!!!");
    trap_handler();
    trap_return();
}
```

且，`trap_return`也不会保存内核现场，只会恢复用户态现场并前往用户态。这样的设计使得trap读起来更清晰，且运行起来成本也更低。