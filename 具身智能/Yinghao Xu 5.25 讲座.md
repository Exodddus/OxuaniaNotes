# LingBot-Map？


# LingBot-World
“没什么Insight，就是要找个大点的厂实习，多搞点卡训模型”

**Data is important**
模型不是智能，数据才是智能
要做漫游长程视频的生成，使用的训练数据有
- real-world long videos
- game videos
- 
![[Pasted image 20260525093404.png]]
特点：动态、一致性
infra和data做到还不错的情况下，就会emerge一些想象不到的能力

# LingBot-VA
什么样的模型是一个好的foundation model？
VLA: 想要复用大模型的能力，但是需要加上与世界交互的功能
模型也会偷懒，容易直接背轨迹
current bottleneck
1. 缺乏动态理解，只有语义推理
2. No memory，可能一直重复某个动作状态


# 双臂具身

追求的目标是用少量的演示轨迹，达成尽量高的成功率

# Q&A
真实的物理很难通过2D的视频来展示
仿真的physics本身就不一定是对的
LingBot-VA的核心insight：自回归框架，具有因果推理能力

为什么视频有卡顿？半异步
VLA成功率能到八九十，但是还没到能够用的状态
可能可以跟传统规划方法做结合？

学校里资源比较少怎么办？scaling law看的不是最后那个点，看的是斜率，所以资源并不是瓶颈
资源多做出来的东西impact比较大，前提是效果比较好，但是并不一定是最优的方法，which应该用小规模实验验证