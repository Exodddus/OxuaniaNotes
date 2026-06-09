***from 机器学习 2026 李宏毅***

# AI Agent 是什么

![[Pasted image 20260402002247.png]]
OpenClaw相当于人和LLM之间的一个接口，“只是一个节肢动物”，并不等于语言模型。
需要时刻记住的一件事情是，语言模型像一个黑盒子，只会根据上文预测下一个token。
 
![[Pasted image 20260402003323.png]]
自己的身份放在prompt里，作为system prompt，在每次调用时都会传入语言模型中
![[Pasted image 20260402003555.png]]

多轮对话中，每次都要把之前的信息作为记忆传入上下文中

# AI Agent 怎么用电脑

有一个特殊的token代表“使用工具”，工具列表及其使用说明会写在system prompt中，如果LLM回传了使用工具的命令，Agent会开始使用这个工具。

![[Pasted image 20260402004406.png]]
OpenClaw强大的原因是可以通过`exec`这个工具来执行shell command，多数时候它都是通过shell来操控电脑的。因此如果agnet读取了网上的信息，可能会对本地带来风险。
可能的防御办法：1. 在prompt中写明需要遵照的规则，但这取决于语言模型遵守指令的能力，不一定好；2. agent每次要执行某个指令时需要人工手动确认
OpenClaw有自己写工具的能力
特殊的工具 SubAgent：繁殖若干个子agent，分别执行不同的任务。这样子大龙虾可以节省context的用量，专注于高阶任务。

*所以龙虾的一个核心任务，就是用尽量少的 context / token 完成需要做的任务。*
如果Sub-agent也能一直繁殖，那会遇到子子孙孙无穷匮，但是没人做任务的问题。所以对于子代需要禁用 Sub-agent能力

# Skill

skill 不等同于工具，但是可以包括使用工具的能力。
OpenClaw 在产生system prompt之前，会在某几个文件夹下搜寻可以用到的skill，然后将skill的说明写入system prompt中
因为上下文是有长度限制的，过早的内容会被存储到MEMORY.md中，以RAG方式检索调用。所以过去的记忆被检索出来时可能不是非常可靠

# Openclaw 的其他小trick
## 心跳机制

每隔一段固定时间，OpenClaw会对语言模型发出一段固定的指令，如 “阅读HEARTBEAT.md，并执行其中的内容”

## cron job 系统
相当于一个排程系统，在某个固定的时间发送一个要求，让LLM学会等待
![[Pasted image 20260402135752.png]]

但是即使是较强的LLM，也不一定能熟练地调用cron job。所以更好的方式是将 cron job 的使用方式写入memory.md里

## Context Compression
memory超过一定长度，就用大模型将其压缩成更短的内容，保存到system prompt里。
如果一个指令没有放到system prompt里，就可能在压缩的过程中丢失。

# 多 Agent 协作

有论文尝试了若干种不同的 Agent 合作框架

![[Pasted image 20260424203814.png]]

![[Pasted image 20260424203954.png]]
* random 是从 Mesh 中剪枝得到的

结论：Chain 方法比较没用，Mesh 和 Random 效果较好，但是不同的任务不一定完全一样

# 多 Agent 社交

LLM 玩狼人杀能出现一些高阶策略。
通过 RL 微调后会玩剧本杀的 LLM，在一些常规任务（数学、指令完成）上表现得更好。

**Moltbook** 上的 AI 发帖有多少人类干预的可能？如果发帖频率较不规律，说明人类干预可能性较大。
深入对话较少，agent 基本只会回一句。

# Agent 辅助学术研究

![[Pasted image 20260424205705.png]]

可以用于写论文、执行实验、设计课题、审查论文等。