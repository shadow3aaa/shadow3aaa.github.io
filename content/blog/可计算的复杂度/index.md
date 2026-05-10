---
title: 可计算的复杂度
date: 2026-05-10T14:49:00+08:00
slug: ''
draft: true
---

## 错误的根源

我认为，vibe coding目前有三层不对齐问题，它们共同导致了工程走向劣化。

第一层是人和代码的不对齐。在传统工程中，代码直接由人的思维逻辑借程序语法产生。因此，bug只可能来自于思维逻辑的漏洞和对语法不了解所导致的错误使用。而在vibe coding中，代码并不直接由人产生，这就导致了人的思维模型和代码实际逻辑产生了新的空隙(即中间用于生成代码的agent)。这空隙最终反映到软件上就产生了bug。

也就是说，第一层不对齐由这两层组成: 人和agent的不对齐，agent和代码的不对齐。

**人和agent的不对齐**来自于过度模糊的意图和反馈的缺乏。比如，“帮我编写一个聊天软件”，这里即使agent完整的产出了软件代码，也会产生巨大的空隙。因为agent无从得知具体应该使用怎么样的技术栈，通信模型等等信息，它只能根据自己的内部参数组合出一套逻辑模型，而人基本没有动力去了解和同步这一大长串代码，这又导致了无法通过反馈来同步逻辑模型。

或者，如果模型能力过弱，直接无法理解人类的逻辑模型，也会导致这种情况，不过这不是工程上可以解决的问题，这里不讨论。

**agent和代码的不对齐**来自于编写工作流本身。举例来说，我有这样一个维护请求计数的源码：

```rust
// gatekeeper.rs
pub struct RequestGate {
    active_requests: usize,
}

impl RequestGate {
    pub fn enter(&mut self) -> Result<(), Error> {
        // 检查限流并增加计数
        if self.active_requests >= 100 { return Err(Error::Busy); }
        self.active_requests += 1;
        Ok(())
    }

    pub fn leave(&mut self) {
        self.active_requests -= 1;
    }
}
```

现在我对 Agent 说：“在进入前增加一个自检模式校验。” Agent 通过 grep 定位到了函数，并读取了代码。它认为在函数开头增加一个错误返回是最优解，于是进行了如下修复:

```diff
--- a/src/gatekeeper.rs
+++ b/src/gatekeeper.rs
@@ -1,5 +1,8 @@
-    pub fn enter(&mut self) -> Result<(), Error> {
+    pub fn enter(&mut self, is_checking: bool) -> Result<(), Error> {
+        if is_checking {
+            return Err(Error::SystemBusy);
+        }
         if self.active_requests >= self.max_limit {
             return Err(Error::TooManyRequests);
         }
```

然而，在它没有注意到的地方，存在这样的逻辑

```rust
gate.enter(is_checking)?; 
do_work();
gate.leave();
```

当 is_checking 为真时，enter 报错返回，但外部调用方的 leave() 依然会被执行，导致 active_requests 计数器下溢。这种修改编译完全通过，但在逻辑上已经由于 Agent 与代码全局上下文的不对齐，埋下了静默崩溃的炸弹。这就是Agent和代码的不对齐所导致的。
