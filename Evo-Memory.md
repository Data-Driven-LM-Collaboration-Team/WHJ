## Evo-Memory Benchmark
区别于传统评估往往只关注模型能否“回忆”起上下文中的静态信息，Evo-Memory 关注的是代理能否通过不断的实践，“吃一堑长一智”，将过去的经验转化为策略，应用到未来的任务中。

## Datasets
A. 单轮推理与问答 (Single-turn Reasoning & QA) ：侧重于评估事实知识和逻辑推理的积累  
MMLU-Pro & GPQA-Diamond : 多学科和研究生级别的科学推理。  
AIME-24/25 : 奥林匹克级别的数学竞赛题，测试符号推理。  
ToolBench : 测试工具使用和 API 调用能力。  

B. 多轮目标导向任务 (Multi-turn Goal-oriented) ： 侧重于评估在交互式环境中的长程规划和探索    
AlfWorld & BabyAI : 具身智能环境中的指令跟随和导航。  
ScienceWorld : 开放式的科学实验模拟。  
Jericho & PDDL : 文本游戏探索和符号规划。

## 评测架构与流程
Evo-Memory 将传统的静态数据集重构为顺序任务流（Sequential Task Streams），且定义了一个统一的 “搜索-合成-进化” (Search-Synthesis-Evolve) 循环 ：  
搜索 (Search): 基于当前输入，从记忆中检索相关条目。  
合成 (Synthesis): 将检索到的信息重组为与当前任务相关的上下文，并生成预测。  
进化 (Evolve): 根据当前的输入、输出和反馈（如任务是否成功），构建新的记忆条目，并更新记忆库。  

所有方法都在该循环下进行评估。
