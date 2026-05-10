---
title: 'SCOPE: Semantic Code Operation & Propagation Engine'
date: 2026-05-10T22:28:00+08:00
slug: semantic-code-engine
draft: true
---

在vibe coding越来越流行的今天，很多拥抱coding agent的项目都面临着同样的问题：明明引入了外部智能体来代理完成项目，却反而导致项目越改越乱。一开始只是些预兆：明明没改过的功能突然报错；修复完一个 bug 又冒出两个；agent修改忘了考虑一处细节。后面变得越来越严重：修复漏洞的速度从一开始的马上发现，变得需要各种调试手段才能勉强完成；实现新feat引发的bug越来越多...

<!--more-->

这不是模型不够聪明。也不是人类描述得不够细。而是 **vibe coding 在人和最终运行的代码之间，楔入了一个全新的间隙，即语言模型本身**。这个间隙带来了两种以前从未被正视过的不对齐。

为此，本文提出SCOPE (Semantic Code Operation & Propagation Engine)来缓解此问题。

## **错误的根源：两层不对齐**

我认为，vibe coding目前有两层不对齐问题，它们共同导致了工程走向劣化。

这两层可以概括为人和代码的不对齐。在传统工程中，代码直接由人的思维逻辑借程序语法产生。因此，bug只可能来自于思维逻辑的漏洞和对语法不了解所导致的错误使用。而在vibe coding中，代码并不直接由人产生，这就导致了人的思维模型和代码实际逻辑产生了新的空隙(即中间用于生成代码的agent)。这空隙最终反映到软件上就产生了bug。

也就是说，vibe coding引入的问题只来自于两类: **人和agent的不对齐**，**agent和代码的不对齐**。

**人和agent的不对齐**来自于过度模糊的意图和反馈的缺乏。比如，“帮我编写一个聊天软件”，这里即使agent完整的产出了软件代码，也会产生巨大的空隙。因为agent无从得知具体应该使用怎么样的技术栈，通信模型等等信息，它只能根据自己的内部参数组合出一套逻辑模型，而人基本没有动力去了解和同步这一大长串代码，这又导致了无法通过反馈来同步逻辑模型。

或者，如果模型能力过弱，直接无法理解人类的逻辑模型，也会导致这种情况，不过这不是工程上可以解决的问题，这里不讨论。

**agent和代码的不对齐**来自于编写工作流本身。假设我们有一个手动管理状态的连接池（这在不方便使用 RAII 的异步状态机或底层 FFI 交互中很常见）：

```rust
// connection_pool.rs
pub struct ConnectionPool {
    active_conns: usize,
    max_limit: usize,
}

impl ConnectionPool {
    /// 获取连接 ID
    pub fn acquire(&mut self) -> Result<u32, Error> {
        if self.active_conns >= self.max_limit { 
            return Err(Error::Busy); 
        }
        
        // 增加计数，并分配底层资源
        self.active_conns += 1;
        let conn_id = unsafe { ffi_allocate_connection() };
        
        Ok(conn_id)
    }

    /// 释放连接 ID
    pub fn release(&mut self, conn_id: u32) {
        unsafe { ffi_free_connection(conn_id); }
        if self.active_conns > 0 {
            self.active_conns -= 1;
        }
    }
}
```

现在，人类提出: “连接时，加一个自检逻辑：如果底层的底层分配失败，请直接返回SystemError，不要把无效的 ID 返回给外部调用方。”

Agent 通过正则或语义搜索定位到了 acquire 函数。敏锐地理解了人类的要求，并在局部做出了看起来极其完美的修改：

```diff
--- a/src/connection_pool.rs
+++ b/src/connection_pool.rs
@@ -5,9 +5,14 @@
         if self.active_conns >= self.max_limit { 
             return Err(Error::Busy); 
         }
         
         self.active_conns += 1;
         let conn_id = unsafe { ffi_allocate_connection() };
         
+        // Agent 增加的自检与拦截逻辑
+        if conn_id == 0 {
+            return Err(Error::SystemError);
+        }
+
         Ok(conn_id)
     }
```

​这段代码编译完全通过，且非常符合人类直觉（拿到无效 ID 就报错）。在局部测试中，它也完全符合人类的要求。​

但实际上，Agent 埋下了一颗静默崩溃的炸弹：由于 Agent 缺乏全局的工程上下文（它没有意识到 self.active_conns += 1; 已经发生在这个提前返回（Early Return）之前），当底层分配失败时，acquire 返回了 Err。由于返回了错误，外部调用方根本拿不到 conn_id，自然也就永远不会调用 release()。

结果就是：每次分配失败，都会导致 active_conns 计数器永久性 +1（状态泄漏）。运行一段时间后，活跃连接数会被虚假的失败请求占满，整个系统陷入死锁，再也无法处理新请求。

## Ripple
