+++
date = '2025-03-20T17:34:41+08:00'
draft = true
title = '[paper] f2fs'
tags = ["SSD", "filesystem", "FAST'15", "FAST"]
+++

- FAST'15
- Samsung Electronics Co., Ltd.


# 1 Motivation

1. random write 很普遍 
2. random write 对 SSD 不友好
- 会影响性能(I/O latency) 
- random write 影响 SSD 寿命

# 2 design
- Flash-friendly on-disk layout
- Cost-effective index structure
- Multi-head logging 
- Adaptive logging
- `fsync` acceleration with roll-forward recovery

## 2.1 On-Disk Layout

![](/img/f2fs-on-disk-layout.png)