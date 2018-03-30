# os作业1

## 列出作业列表，计算可调度性

1. OS 调研报告  take: 4days remain: 6days
2. 其他作业 take 1.5days remain: 6days
3. 视频剪辑 take 10days remain: 60days
4. 计网服务器设计 take 4days remain 10days

可调度性： 4/6 + 1.5/6 + 10/60 + 4/10 = 1.483 有可行调度方案

调度方案

os 调研报告

其他作业

服务器设计

视频剪辑

## 任务切换的伪代码 task1->task2->task1

push retAddr1 ;保存返回task1的地址

Save Registers ;task2 负责保存寄存器 callee save

carryout task2

restore Registers ; 恢复寄存器状态

ret ;回到task1