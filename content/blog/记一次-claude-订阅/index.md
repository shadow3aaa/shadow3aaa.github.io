---
title: 记一次 Claude 订阅
date: 2026-10-28T20:44:00+08:00
slug: how-do-I-get-claude
draft: true
---

最近买了 gpt 的 pro 订阅（5x挡位的），发现这玩意虽然天天重置怎么还是用的那么快。正好今天忙着谈恋爱的 tibo 又又又在他那28day计划里没活了进行重置，对额度总量进行一个测算。

codex 能直接读取重置以来使用的 token 数量，考虑到我仅用了6.1sol，应该还是挺准的。他的输出原文如下：

```text
按本周重置以来的完整本地记录反推，周额度约为 13.19 亿 tokens。
项目	数值
重置以来已用 tokens	422,110,242（约 4.22 亿）
当前已用额度	32%
推算周总量：已用 ÷ 32%	约 13.19 亿
剩余 68% 对应量	约 8.97 亿


统计从 **10 月 28 日 14:46（北京时间）**开始，覆盖 6 个本地会话。这次包含昨天重置后至今的完整用量。

这是按当前使用方式折算的等效额度：其中约 98.3% 的输入命中缓存。如果模型、缓存比例或输出比例变化，能使用的实际 token 总量也会变化；它不是官方固定 token 配额。
```

也就是说一周仅有 1.3B 的 token .... 令人忍俊不禁。

而隔壁 claude pro （注意不是100$的 claude max 5x） 仅用能力类似（当然前端 opus5.5 比 6.1sol 强的多）的 opus5.5，根据社区调查，一周额度大概在8-13亿之间。也就是说无论如何，两个 claude pro 的额度就能把 gpt pro 5x 吊起来当陀螺抽。注意到 `20$ * 2 > 100$`，没有人能绷的住。这下不得不跑路了。

## 方案

经过调查，我认为最便宜稳定的方案是搞个 us 的 vps，然后 vps 连上便宜干净 isp 代理。（这里用us的vps并非必须，只是为了和 isp 连接延迟低，并且方便注册等等具备一个 us 环境。理论上可以直接惬意跑任意设备上，反正ip又不走 vps 的机房ip）。然后购买稳定的google账号，使用礼品卡充值，再在安卓端订阅。

## 购买 VPS

这里刚好 [cloudcone 在打折](https://hello.cloudcone.com/halloween-vps-sale-2026/)。购买他们的 Frankenstein-26-SSD-VPS-1 机器，一年仅需 20.25$。虽然内存仅 1.25g ，但是对反代绰绰有余了。

我使用国内的 mastercard 购买的，不过他们支持支付宝，并未尝试是否可以正常使用。开好机器后，他们会给你发邮件，里面包含vps的密码。

## 关于 IP

IP 这边的方案我没有直接购买家宽的 vps，因为这种 vps 的价格看了直接吓哭了。举例说 LisaHost 的 2g 内存挡大概最低的线路也要 $13 每月。诚然，这种家宽 vps 的 ip 大概是最干净的，然而也贵太多了。

我选择的是购买 [IPRoyal 的 ISP 代理](https://iproyal.cn/isp-proxies/)。只要运气不太差，写好对 ip 的要求，还是可以得到不错的 ip 的。以下是详细的建议操作步骤

1. 选择购买 ISP 代理
2. 代理类型（CHOOSE PROXY TYPE）选择 Dedicated proxies（独享代理）。独享 IP 是防封的基础，不要选 Shared。
3. 订阅周期（SELECT SUBSCRIPTION PLAN）按需选择，考虑到订阅的特性，一半选 30 days。
4. 代理地区（CHOOSE LOCATION）：选择 United States。当然。
5. 欺诈评分（CHOOSE FRAUD SCORE）：选择Premium。这个会要求多支付 35% 的费用，但是显而易见应该选择。
5. EXTRAS **重点！** 强烈建议勾选Add extra requirements。需要多支付 30% 费用。但是你可以人工写对 ip 的要求。这能得到更好的 IP。具体的要求可以参考下面的。

```text
Target: Clean US Residential/ISP IP.
Requirements: Native status on Ping0.cc & IPinfo; IP2Location: ISP Fixed Line, valid US domain; Scamalytics Fraud Score < 15.
```

值得注意的是，IPRoyal 的代理24小时内可以免费要求重新换 ip 。拿到之后建议用各种检测网站检测下。如果发现问题可以找客服换，别浪费这个更换机会。

算上这些费用增加，一个月也才 7$。支付方式似乎也支持支付宝，不过我依旧是mastercard进行的支付。似乎因为选择了额外要求，订单处理的比较慢。等了半小时才得到代理。

## 配置

在 VPS 上配置好代理（推荐直接sing-box走全局了，反正反代没啥大流量也没什么带宽需求）。
