***VLingNav: Embodied Navigation with Adaptive Reasoning and Visual-Assisted Linguistic Memory***
## pipeline:

- AdaCoT - 根据环境复杂程度自动决定是否开启长思维链
    
- VLingMem - 用文本形式描述CoT中的关键视觉信息，并压缩储存在上下文中
    

## data collection

- Nav-AdaCoT-2.9M
    
    - 任务范围：ObjectNav EVT(具身视觉跟踪) ImageNav
        
    - 包含内容：290万个动作样本，47.2 万条思维链标注，自适应标签(think_on, think_off)
        
    - 自动化标注流程
        
## training recipe

- pre-train
    
- SFT
    
    - 将 290 万条具身导航数据（包括 ObjectNav、EVT、ImageNav）与 160 万条开放世界视频数据（如视频问答）混合在一起进行训练
        
    - 模型会学习专家提供的轨迹标签。每当输入一段视频流，模型会尝试预测接下来的运动路径。系统会计算预测结果与专家真实动作（Ground-truth）之间的误差，从而让机器人的移动向专家路径靠拢
        
    -  利用 **Nav-AdaCoT-2.9M** 数据集中的 47.2 万条思维链标注，模型在这一阶段学会了如何在脑海中生成“解题思路”。它会学习在特定环境下生成 `<think>` 推理内容和 `<summary>` 环境总结。这些文本输出通过交叉熵损失（CE Loss）进行监督，使模型的思考逻辑与专家标注的一致。
        
    - 思维链自适应触发：这一步还教会了模型根据当前环境的复杂程度，自主决定是否需要开启推理模式。通过学习数据集中预设的 `<think_on>` 和 `<think_off>` 标签，模型开始掌握在简单路径下快速行动、在复杂分岔路口停下来思考的本领。
        
- Online Expert-guided Post-training
    
    - 概率连续动作模型 - 基于MLP设计概率投射头，将VLM输出的特征h_t映射为一个多元高斯分布，给出动作均值和标准差（的对数）的预测。在训练阶段，模型从这个高斯分布中随机采样动作；在测试阶段，取均值作为动作输出
        
    - 混合回放策略 - Native Rollout 自主探索，只把成功的轨迹放入缓冲区；Expert-guided Rollout 专家引导错误纠正 当机器人遇到异常时启用专家策略，演示脱困路径，纠错数据也会进入缓冲区
        
    - 带有增强损失函数的在线微调 - 为解决奖励稀疏问题，损失函数由两部分混合而成——RL损失与SFT损失。其中SFT损失的计算方式继承了上一部分的监督微调损失，即结合动作预测损失和文本输出损失
        

Comment: Project page: https://wsakobe.github.io/VLingNav-web/