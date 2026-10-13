# 昇腾 NPU 网络诊断采集 — 192.168.0.99（2026-10-13）

按华为支持要求执行的 hccn_tool 网络诊断 + 官方日志采集，供运维/售后分析 RoCE 链路问题。

## 目标服务器

| 项 | 值 |
|---|---|
| 私网 IP | 192.168.0.99 |
| hostname | aura-1.novalocal（注意：集群内另有同名克隆机，勿混淆） |
| NPU | 8 × 昇腾（npu-smi Total Count: 8） |
| 采集时间 | 2026-10-13 11:07–11:13 CST |

## 关键发现（hccn.log 摘要）

- **NPU 3（devid 3）当前 link DOWN**，net_health 对应 **"Receive timeout"**；其余 7 卡全部 UP / Success。
- link_stat 历史：**devid 3 于 2026-10-13 04:04:46 再次 LINK DOWN**（本次报障的直接线索）。
- 全部 8 卡在 **2026-08-14 23:26–23:27** 有同批 LINK DOWN 记录（各 1 次，疑似一次整机断电/重启，devid 3 因今日事件累计 2 次）。

> 结论指向：3 卡网口/光模块/线缆/交换机端口单点异常，建议优先交叉验证。

## 文件清单

| 文件 | 位置 | 说明 |
|---|---|---|
| `hccn.log` | 本仓库 | hccn_tool 七项诊断输出（link / link_stat / net_health / lldp / optical / scdr / serdes，devid 0–8），网页可直接查看 |
| `hccn_diag_192.168.0.99_20261013.tar.gz` | [Release 附件](../../releases)（92 MB） | 内含官方 `npu_log_collect.sh` 产物 `npu_log_collect_20261013111314.tar.gz` + hccn.log 原件 |

服务器原件路径：`192.168.0.99:/root/hccn_diag_20261013/`

## 采集命令（修正版）

原始指令后 4 个循环缺少 `done`，以下为修正后实际执行的版本：

```bash
cd /root/hccn_diag_20261013
for i in $(seq 0 8); do echo "$i"; hccn_tool -i $i -link -g; done >> hccn.log 2>&1
for i in $(seq 0 8); do hccn_tool -i $i -link_stat -g; done >> hccn.log 2>&1
for i in $(seq 0 8); do echo "***$i***"; hccn_tool -i $i -net_health -g; done >> hccn.log 2>&1
for i in $(seq 0 8); do echo "id=${i}"; timeout 10s hccn_tool -i $i -lldp -g; done >> hccn.log 2>&1
for i in $(seq 0 8); do echo "id=${i}"; timeout 10s hccn_tool -i $i -optical -g; done >> hccn.log 2>&1
for i in $(seq 0 8); do echo "id=${i}"; timeout 10s hccn_tool -i $i -scdr -t 5; done >> hccn.log 2>&1
for i in $(seq 0 8); do echo "id=${i}"; timeout 10s hccn_tool -i $i -serdes -g; done >> hccn.log 2>&1
/usr/local/Ascend/driver/tools/npu_log_collect.sh   # 产物：npu_log_collect_*.tar.gz
```

注意：`npu_log_collect.sh` 以 `ps | grep 自身脚本名` 判断重复实例——若 ssh 命令行中含该字符串会误报 "Other task is running"，需以 `bash -s` 流式执行。
