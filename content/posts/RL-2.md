---
title: "强化学习(2)"
date: 2022-10-20T15:35:47+08:00
draft: true
math: true
tags: ["ML", "强化学习"]
summary: 强化学习学习笔记
---

## Actor-Critic

+ 状态价值函数定义

$V(s;\theta,w)=\sum_{a}\pi(a|s;\pi)\cdot q(s,a;w)$

定义策略网络$\pi(a|s;\pi)$最大化$V$, 定义价值网络优化评估结果$q$近似度.

![](/images/qdn-3.png)

