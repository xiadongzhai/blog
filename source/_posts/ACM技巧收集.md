---
title: ACM技巧收集
date: 2026-11-09 21:36:30
tags:
---

一，sort
1.      // 自定义比较器：偶数在前，同奇偶时数值升序 
        sort(a.begin(), a.end(), [](int u, int v) { 
             int pu = u & 1, pv = v & 1;    // 取奇偶性：0 表示偶数，1 表示奇数 
            if (pu != pv) return pu < pv;  // 偶数(0) < 奇数(1)，所以偶数排在前面 
            return u < v;                  // 奇偶性相同，按数值从小到大 
        }); 