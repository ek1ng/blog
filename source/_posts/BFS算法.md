---
title: BFS
date: 2021-08-10 00:00:00
tags: [leetcode,algorithm]
---
# BFS算法

**声明：文章内容为作者阅读[labuladong的算法笔记](labuladong.gitee.io/)后的个人笔记分享**

## 概述

BFS 的核心思想是把一些问题抽象成图，从一个点开始，向四周开始扩散。让你在一幅「图」中找到从起点 `start` 到终点 `target` 的最近距离。一般来说，我们写 BFS 算法都是用「队列」这种数据结构，每次将一个节点周围的所有节点加入队列。

## 框架

```java
// 计算从起点 start 到终点 target 的最近距离
int BFS(Node start, Node target) {
    Queue<Node> q; // 核心数据结构
    Set<Node> visited; // 避免走回头路
    
    q.offer(start); // 将起点加入队列
    visited.add(start);
    int step = 0; // 记录扩散的步数

    while (q not empty) {
        int sz = q.size();
        /* 将当前队列中的所有节点向四周扩散 */
        for (int i = 0; i < sz; i++) {
            Node cur = q.poll();
            /* 划重点：这里判断是否到达终点 */
            if (cur is target)
                return step;
            /* 将 cur 的相邻节点加入队列 */
            for (Node x : cur.adj())
                if (x not in visited) {
                    q.offer(x);
                    visited.add(x);
                }
        }
        /* 划重点：更新步数在这里 */
        step++;
    }
}
```

