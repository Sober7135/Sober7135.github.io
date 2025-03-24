+++
date = '2025-03-24T02:34:01+08:00'
draft = true
title = 'Notes for Linux Kernel Blk Mq'
+++

TODO

[Hardware dispatch queues](https://docs.kernel.org/block/blk-mq.html#hardware-dispatch-queues)
> The hardware queue (represented by struct blk_mq_hw_ctx) is a struct used by device drivers to map the device submission queues (or device DMA ring buffer), and are the last step of the block layer submission code before the low level device driver taking ownership of the request. To run this queue, the block layer removes requests from the associated software queues and tries to dispatch to the hardware.
> If it’s not possible to send the requests directly to hardware, they will be added to a linked list (hctx->dispatch) of requests. Then, next time the block layer runs a queue, it will send the requests laying at the dispatch list first, to ensure a fairness dispatch with those requests that were ready to be sent first. The number of hardware queues depends on the number of hardware contexts supported by the hardware and its device driver, but it will not be more than the number of cores of the system. There is no reordering at this stage, and each software queue has a set of hardware queues to send requests for.

以上是 Linux Kernel Docs 中的内容

1. 实质上, `blk_mq_hw_ctx` 表示就是 hardware queue, 用于 mapping the device submission queues (比如 NVMe 中的 sq)
2. requests 不能直接发给 hardware, 
   1. 首先要加到 hctx->dispatch (linked list of requests), 
   2. 
