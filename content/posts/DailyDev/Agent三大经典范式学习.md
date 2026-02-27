---
title: "Agent三大经典范式学习"
date: 2026-02-27
aliases: ["/Daily Dev"]
tags: ["Agent", "AI"]
categories: ["Daily Dev"]
author: "Kurong"
showToc: true
TocOpen: true
draft: false
hidemeta: false
comments: false
description: ""
disableHLJS: false # to disable highlightjs
disableShare: true
disableHLJS: false
hideSummary: false
searchHidden: false
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true

---

当前大模型智能体（LLM-based Agents）设计中最为核心的三种认知范式。

- ReAct (Reasoning + Acting)
- Plan-and-Solve (P&S)
- Self-Reflection (Reflexion)

## 1. ReAct (Reasoning + Acting)

### 核心概念

ReAct 由 Google 和普林斯顿大学提出。其核心思想是让大模型交替进行**推理（Thought）\**和\**行动（Action）**。

- **Thought**: 帮助模型明确当前目标、分析环境、追踪状态。
- **Action**: 允许模型通过工具（如搜索引擎、计算器）与外部环境交互。
- **Observation**: 观察行动的结果并将其反馈给模型。

### 流程细节

1. **接收任务**: 输入用户指令。
2. **推理循环**:
   - 模型生成 `Thought`：分析当前进度，决定下一步做什么。
   - 模型生成 `Action`：指定调用的工具和参数。
   - 环境执行 `Action` 并返回 `Observation`。
   - 将 `Thought`, `Action`, `Observation` 全部加入上下文，循环直到任务完成。
3. **输出**: 最终生成 `Final Answer`。

### 伪代码实现 (Python)

```python
import logging
from typing import List, Dict, Any

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("ReActAgent")

class Tool:
    """Mock tool class for demonstration"""
    def call(self, action_input: str) -> str:
        # Simulate tool execution
        return f"Result of {action_input}"

class ReActAgent:
    def __init__(self, model, tools: Dict[str, Tool]):
        self.model = model
        self.tools = tools
        self.max_steps = 5

    def run(self, query: str) -> str:
        """
        Execute the ReAct loop: Thought -> Action -> Observation
        """
        context = f"Question: {query}\n"
        
        for step in range(self.max_steps):
            logger.info(f"Step {step + 1}: Generating Thought and Action")
            
            # 1. Generate Thought and Action based on current context
            response = self.model.generate(context + "Thought:")
            
            if "Final Answer:" in response:
                logger.info("Task completed.")
                return response.split("Final Answer:")[-1].strip()

            # 2. Parse action (Assume format: Action: [tool_name], Action Input: [input])
            try:
                tool_name, action_input = self._parse_action(response)
                logger.info(f"Executing Action: {tool_name} with input: {action_input}")
                
                # 3. Execute tool and get observation
                observation = self.tools[tool_name].call(action_input)
                
                # 4. Update context for the next iteration
                context += f"{response}\nObservation: {observation}\n"
            except Exception as e:
                logger.error(f"Failed to execute action: {e}")
                context += f"\nError: {str(e)}. Try a different approach.\n"

        return "Failed to reach a conclusion within max steps."

    def _parse_action(self, text: str) -> (str, str):
        # Implementation of parsing logic (regex or string split)
        # Placeholder logic
        return "Search", "Python Agent patterns"
```

## 2. Plan-and-Solve (P&S)

### 核心概念

Plan-and-Solve 旨在解决 Zero-shot Chain-of-Thought (CoT) 在处理复杂任务时容易“跑偏”或遗漏步骤的问题。它将决策分为两个阶段：

1. **Planning**: 将复杂任务分解为一系列小的子任务（子计划）。
2. **Solving**: 按照计划逐一执行子任务。

### 流程细节

1. **计划阶段**: 提示模型“让我们先制定一个计划”，生成步骤列表 $S_1, S_2, ..., S_n$。
2. **执行阶段**:
   - 维持一个状态跟踪器。
   - 遍历每个步骤，调用模型或工具解决该特定步骤的问题。
   - 汇总每步的结果得到最终解。

### 伪代码实现 (Python)

```python
from typing import List
import logging

logger = logging.getLogger("PlanAndSolveAgent")

class PlanAndSolveAgent:
    def __init__(self, model):
        self.model = model

    def run(self, complex_task: str) -> str:
        """
        Two-stage process: Plan then Execute
        """
        # Phase 1: Planning
        logger.info("Phase 1: Generating global plan")
        plan_prompt = f"Task: {complex_task}\nBreak this down into logical steps."
        plan_raw = self.model.generate(plan_prompt)
        steps = self._extract_steps(plan_raw)

        # Phase 2: Execution
        logger.info(f"Phase 2: Executing {len(steps)} steps")
        results = []
        for i, step in enumerate(steps):
            logger.info(f"Executing step {i+1}: {step}")
            execution_prompt = f"Task: {complex_task}\nPlan: {plan_raw}\nNow, execute Step {i+1}: {step}"
            step_result = self.model.generate(execution_prompt)
            results.append(step_result)

        # Phase 3: Synthesis
        logger.info("Synthesizing final answer")
        final_prompt = f"Based on these steps: {results}, provide the final answer."
        return self.model.generate(final_prompt)

    def _extract_steps(self, plan_text: str) -> List[str]:
        # Simple split logic for demo
        return [s.strip() for s in plan_text.split('\n') if s.strip()]
```

## 3. Self-Reflection (Reflexion)

### 核心概念

Self-Reflection 引入了“闭环反馈”机制。Agent 不再是一次性输出，而是会审视自己的解答，寻找错误，并进行自我修正。典型的框架如 **Reflexion**：

- **Actor**: 生成尝试。
- **Evaluator**: 打分或判断成功/失败。
- **Reflector**: 分析失败原因，生成“经验教训”存入长期记忆。

### 流程细节

1. **初次尝试**: Actor 执行任务。
2. **评估**: Evaluator 判断输出是否符合预期。
3. **反思**: 如果失败，模型会根据（输入、失败输出、奖励信号）生成一段文字形式的反思。
4. **迭代**: 模型在下一次尝试时，会将之前的反思作为额外上下文，避免重蹈覆辙。

### 伪代码实现 (Python)

```python
import logging
from typing import Optional

logger = logging.getLogger("ReflectionAgent")

class ReflectionAgent:
    def __init__(self, actor_model, reflector_model):
        self.actor = actor_model
        self.reflector = reflector_model
        self.memory = [] # Store past reflections

    def run(self, task: str, max_iterations: int = 3) -> str:
        """
        Iterative loop: Attempt -> Evaluate -> Reflect -> Improve
        """
        current_attempt = ""
        
        for i in range(max_iterations):
            logger.info(f"Iteration {i+1}: Attempting task")
            
            # Include past reflections in context
            reflection_context = "\n".join(self.memory)
            prompt = f"Task: {task}\nPast lessons: {reflection_context}\nAnswer:"
            
            current_attempt = self.actor.generate(prompt)
            
            # Evaluate (In real scenarios, use unit tests or another LLM)
            is_correct, feedback = self._evaluate(current_attempt)
            
            if is_correct:
                logger.info("Evaluation passed.")
                return current_attempt
            
            # Generate reflection on failure
            logger.warning(f"Attempt failed. Generating reflection. Feedback: {feedback}")
            reflection_prompt = f"Task: {task}\nFailed Answer: {current_attempt}\nFeedback: {feedback}\nWhat went wrong?"
            reflection = self.reflector.generate(reflection_prompt)
            
            # Store learning in memory
            self.memory.append(f"Attempt {i+1} Lesson: {reflection}")

        return current_attempt

    def _evaluate(self, result: str) -> (bool, str):
        # Mock evaluation logic
        # Returns (Success, Feedback string)
        if "Correct Keyword" in result:
            return True, "Perfect"
        return False, "Missing specific technical details"
```

## 总结对比

| 范式                | 核心侧重点         | 适用场景                                 | 局限性                                    |
| ------------------- | ------------------ | ---------------------------------------- | ----------------------------------------- |
| **ReAct**           | 实时交互、动态调整 | 搜索、数据库查询、需要频繁外部反馈       | 步数多时上下文开销大，容易陷入无限死循环  |
| **Plan-and-Solve**  | 结构化拆解、全局观 | 复杂数学题、多步骤数据处理、长文档生成   | 初始计划如果错误，后续执行可能失效        |
| **Self-Reflection** | 自我优化、错误校正 | 编程任务、需要逻辑严密性的推理、多轮博弈 | 依赖 Evaluator 的准确性，多次调用成本较高 |