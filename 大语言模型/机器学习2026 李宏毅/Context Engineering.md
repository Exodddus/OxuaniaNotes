***from 机器学习 2026 李宏毅***

![[Pasted image 20260409004259.png]]
Context Engineering 就是那个 F 函数

进阶版：![[Pasted image 20260409012215.png]]

# 压缩
- 最简单的方式：调用大模型将上文概括为 summary
- observation masking: 把工具输出改成 “这里曾经有个工具的输出”。因为这样可以避免反复调用同一个工具
	- arxiv.org/abs/2508.21433
- 可以同时使用两者，在前期先采用 observation masking，后期memory太多了再进行直接概括。

> context vs prompt：context 只有一部分会被放进prompt中，另一部分存在硬盘里

ACON - 2510.00615：将出现问题的摘要（包括摘要前后的内容）喂给另一个LLM，让它总结经验，将经验feedback作为prompt输入给摘要LLM优化结果，发现acc和token消耗都更好了。
也可以用强化学习特化训练模型摘要能力。

## 什么时候开始压缩？

Openclaw 写死了压缩的规则，因为语言模型自己不喜欢选择压缩
![[Pasted image 20260409010347.png]]

subagent也可以被视作一种自主压缩方式，因为其会预示接下来的一段context会被压缩为返回的结果。
强化学习训练可以优化产生sub-agent的能力，但是除了结果正确性还要加上“主干不能过长”“不能做出超出范围的事情”等reward。

# 过滤

![[Pasted image 20260409010738.png]]
两篇论文共同结论：多数context花在阅读文件
因此在阅读时就可以过滤出有效内容，降低context消耗。
![[Pasted image 20260409010911.png]]

Openclaw设计了memory_get这个特别的工具用于读取memory所需要的部分

- 按需加载
MCP-Zero - 2506.01056
![[Pasted image 20260409011322.png]]

# Agentic Context Engineering
让LLM自主决定如何压缩上下文

![[Pasted image 20260409011710.png]]

另一种思路是利用LLM写代码的能力，只输入 context t 的元数据，让它取Hard Disk里搜索需要的信息（类似RAG），再对meta data进行修改。

![[Pasted image 20260409011832.png]]
这种方法可以提升在非常长的context上的任务表现

