+++
date = '2025-03-26T22:18:39+08:00'
draft = false
title = '如何在 FEMU 中添加 NVMe CMD'
tags = ["SSD", "Emulator", "FEMU", "NVMe"]
+++

水一篇

# Background
---

## 什么是 FEMU

FEMU 是一个基于 QEMU 的 SSD Emulator, 发表在 FAST'18
- [paper](https://www.usenix.org/system/files/conference/fast18/fast18-li.pdf)
- [code](https://github.com/MoatLab/FEMU)

## 什么是 NVMe 协议
[维基百科](https://zh.wikipedia.org/wiki/NVM_Express)[^1]
>NVM Express（缩写NVMe），或称非易失性内存主机控制器接口规范（英語：Non-Volatile Memory Host Controller Interface Specification，缩写：NVMHCIS），是一个逻辑设备接口规范。

[NVM Express® Specification Family](https://nvmexpress.org/specifications/)[^2]
![](/img/nvme-family-of-specification.png)

可以看到 NVMe specification 主要分为以上这几部分, 本文讨论的最基础的部分, 不涉及 ZNS, KV, RDMA...等, 即 FEMU 中最基础的 blackbox mode.

## NVMe 中的 COMMAND 
笔者了解也不是很全面, 只是简单介绍一下 I/O Commands 和 Admin Command [^3]

> The Admin Command Set defines the commands that may be submitted to the Admin Submission Queue.
> An I/O command is a command submitted to an I/O Submission Queue.

可以简单理解为, Admin Command 和 I/O Command 的区别是提交的位置不同
- I/O Command 可以简单理解为 I/O 相关的, 比如 Write/Read/Fluash...
- Admin 顾名思义就是管理类的, 比如管理 I/O Submission/Completion Queue

I/O Command 可以根据 I/O Command Set 继续细分, e.g. NVM, Key Value, Zoned Namespace, 这里我们只关注 NVM

这里我们是需要添加 Command, 所以只需要关注 Vendor specific 部分就可以了

Opcodes for Admin Commands:
<table border="1">
  <thead>
    <tr>
      <th colspan="2">Opcode by Field</th>
      <th rowspan="2">Combined<br>Opcode</th>
      <th rowspan="2">Namespace<br>Identifier Used</th>
      <th rowspan="2">Command</th>
      <th rowspan="2">Reference</th>
    </tr>
    <tr>
      <th>(07:02)<br>Function</th>
      <th>(01:00)<br>Data Transfer</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <td>0011 01b</td>
      <td>01b</td>
      <td>35h</td>
      <td>No</td>
      <td>Manage Exported Port</td>
      <td>5.3.9</td>
    </tr>
    <tr>
      <td>0011 10b</td>
      <td>01b</td>
      <td>39h</td>
      <td>No</td>
      <td>Send Discovery Log Page</td>
      <td>5.3.10</td>
    </tr>
    <!-- add more rows as needed -->
    <tr>
      <td colspan="6" style="text-align:center; font-weight:bold;">Vendor Specific</td>
    </tr>
    <tr>
      <td>11xx xxb</td>
      <td>NOTE 3</td>
      <td>C0h to FFh</td>
      <td></td>
      <td>Vendor specific</td>
      <td></td>
    </tr>
  </tbody>
</table>

如果我们需要添加一条自定义的 command 按照 Vendor Specific 的格式即可

# 如何在 FEMU 中实现?

先说一下我的需求, 需求是记录一段时间的 latency, 这样我们就需要两个 command
1. 用于 clear 当前的结果
2. 用于 report 特定时间的 latency

出于简单考虑, 直接就选择 Admin, CMD 的结果呈现在 LOG 中
 
添加 opcode
```C
enum NvmeAdminCommands {
    // ...
    NVME_ADM_CMD_SECURITY_SEND  = 0x81,
    NVME_ADM_CMD_SECURITY_RECV  = 0x82,
    NVME_ADM_CMD_SET_DB_MEMORY  = 0x7c,
+   NVME_ADM_CMD_PRINT_LATENCY  = 0xc0,
+   NVME_ADM_CMD_CLEAR_LATENCY  = 0xc1,
    NVME_ADM_CMD_FEMU_DEBUG     = 0xee,
    NVME_ADM_CMD_FEMU_FLIP      = 0xef,
};
```

在 `ssd_read`/`ssd_write` 后, 更新我们需要统计的信息
```C
            switch (req->cmd.opcode) {
            case NVME_CMD_WRITE:
                lat = ssd_write(ssd, req);
+               lat_stat_update(&n->write_lat_stat, lat);
                break;
            case NVME_CMD_READ:
                lat = ssd_read(ssd, req);
+               lat_stat_update(&n->read_lat_stat, lat);
                break;
            case NVME_CMD_DSM:
                lat = 0;
                break;
            default:
                //ftl_err("FTL received unkown request type, ERROR\n");
                ;
            }
```

需要处理新添加的 opcode, 这里出于简单考虑直接用 log 输出

如果需要正经得到结果可能需要添加 I/O command, 然后还得修改 nvme-cli 源代码来从 memory display 出这个信息

```C
static uint16_t bb_admin_cmd(FemuCtrl *n, NvmeCmd *cmd)
{
    switch (cmd->opcode) {
    case NVME_ADM_CMD_FEMU_FLIP:
        bb_flip(n, cmd);
        return NVME_SUCCESS;
+   case NVME_ADM_CMD_PRINT_LATENCY:
+       femu_log( "print latency statistic: "); 
+       femu_log("\t[write]: avg=%lf, stdev=%lf, count=%lu", n->write_lat_stat.mean, 
+           sqrt(n->write_lat_stat.m2 / n->write_lat_stat.count), n->write_lat_stat.count);
+       femu_log("\t[read]: avg=%lf, stdev=%lf, count=%lu", n->read_lat_stat.mean, 
+           sqrt(n->read_lat_stat.m2 / n->read_lat_stat.count), n->read_lat_stat.count);
+       return NVME_SUCCESS;
+   case NVME_ADM_CMD_CLEAR_LATENCY:
+       femu_log( "clear latency statistic.");
+       memset(&n->read_lat_stat, 0, sizeof(LatencyStat));
+       memset(&n->write_lat_stat, 0, sizeof(LatencyStat));
+       return NVME_SUCCESS;
    default:
        return NVME_INVALID_OPCODE | NVME_DNR;
    }
}
```

## 运行

- `sudo nvme admin-passthru /dev/nvme0 --opcode=0xc0` 打印 latency 统计信息
- `sudo nvme admin-passthru /dev/nvme0 --opcode=0xc1` 清空 latency 统计信息

结果类似这样

```
2024-10-22 19:43:14 [FEMU] ../hw/femu/bbssd/bb.c:104 bb_admin_cmd Log: print latency statistic: 
2024-10-22 19:43:14 [FEMU] ../hw/femu/bbssd/bb.c:105 bb_admin_cmd Log:  [write]: avg=371867.940275, stdev=3076658.698984, count=1296538
2024-10-22 19:43:14 [FEMU] ../hw/femu/bbssd/bb.c:107 bb_admin_cmd Log:  [read]: avg=229969.164519, stdev=3101376.001435, count=1945039
```

# Reference

[^1]: https://zh.wikipedia.org/wiki/NVM_Express
[^2]: https://nvmexpress.org/specifications/
[^3]: NVM Express Base Specification, 1.5 Definition, https://nvmexpress.org/specifications/
