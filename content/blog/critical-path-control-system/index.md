---
title: Critical-Path Control System[draft]
date: 2026-02-25T19:46:00+08:00
slug: the-critical-path-control-system
draft: true
---

本文记录CPCS的研究

<!--more-->

## 介绍

CPCS，即Critical-Path Control System，是FAS(Frame Aware Scheduling)的补充。其目的在于分析关键线程路径，以更合理的分配FAS当前分配的总性能到各集簇。

## POC

首先需要验证可行性。

没有人可以未卜先知，因此无法在本帧执行之前就得知本帧的关键线程路径是什么。所以，CPCS有效的前提是游戏帧的行为**高度重复。**如果这点成立，就可以用之前帧的关键路径预测本帧的关键路径。这就是本次POC要验证的目标。

根据我之前（[shadow3aaa/frame-analyzer-ebpf](https://github.com/shadow3aaa/frame-analyzer-ebpf)）的经历，ebpf的编写和使用堪称折磨，即使是使用了aya-rs这种高度简化构建流程的框架。所幸我可以让codex帮我完成poc程序的编写，只需要抄frame-analyzer的代码即可。

POC的结构如下。

```mermaid
flowchart TD
      subgraph Kernel[eBPF / Kernel]
          K1[Emit Event Packet]
      end

      subgraph UserSpace[Analyzer / User Space]
          K1 --> U1[接收事件流]
          U1 --> U2{frame_point?}

          U2 -- 否 --> U3[写入当前帧桶]
          U3 --> U4[更新关联状态<br/>pending_wait / latest_wake / wakeup->running]
          U4 --> U1

          U2 -- 是 --> U5[分析上帧数据]
          U5 --> U6[构建关系图<br/>futex边 + sched边]
          U6 --> U7[记录关键依赖链]
          U7 --> U8[开始下一帧]
          U8 --> U1
      end
```

测试了游戏场景，得到下面这个dag图。红色即关键路径。

![线程依赖图](cpcs_overlay.png)

可以看到从线程(tid3575)开始的红色路径是相对集中的。因此确实有可行性。
