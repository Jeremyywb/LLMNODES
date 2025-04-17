# RLHF 训练代码



> 以下代码由Claude ai生成，准确性未验证，

```python

import torch
import torch.nn as nn
import torch.nn.functional as F
import torch.optim as optim
from torch.distributions import Categorical
from transformers import AutoModelForCausalLM, AutoTokenizer
import numpy as np
from tqdm import tqdm
from dataclasses import dataclass
from typing import List, Tuple, Dict, Optional
import wandb

# 配置参数
@dataclass
class RLHFConfig:
    pretrained_model: str = "gpt2"  # 预训练模型
    lr: float = 1e-5  # 学习率
    batch_size: int = 8  # 批次大小
    epochs: int = 3  # 训练轮数
    max_seq_len: int = 512  # 最大序列长度
    kl_coef: float = 0.1  # KL散度系数
    discount_factor: float = 0.99  # 折扣因子
    gae_lambda: float = 0.95  # GAE lambda系数
    ppo_epochs: int = 4  # PPO更新轮数
    ppo_clip: float = 0.2  # PPO裁剪系数
    value_clip: float = 0.2  # 价值裁剪系数
    entropy_coef: float = 0.01  # 熵系数
    device: str = "cuda" if torch.cuda.is_available() else "cpu"

# 第一阶段：监督微调
def supervised_fine_tuning(model, tokenizer, train_dataset, config):
    """
    使用人类编写的高质量回答进行监督微调
    """
    optimizer = optim.AdamW(model.parameters(), lr=config.lr)
    model.train()
    
    for epoch in range(config.epochs):
        total_loss = 0
        for batch in tqdm(train_dataset.batch(config.batch_size), desc=f"SFT Epoch {epoch}"):
            # 准备输入数据
            inputs = tokenizer(batch["prompt"], return_tensors="pt", padding=True, truncation=True, 
                              max_length=config.max_seq_len).to(config.device)
            labels = tokenizer(batch["response"], return_tensors="pt", padding=True, truncation=True,
                              max_length=config.max_seq_len).to(config.device)
            
            # 前向传播和损失计算
            outputs = model(input_ids=inputs.input_ids, 
                           attention_mask=inputs.attention_mask,
                           labels=labels.input_ids)
            loss = outputs.loss
            
            # 反向传播和优化
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
            
            total_loss += loss.item()
        
        print(f"Epoch {epoch}, Average Loss: {total_loss / len(train_dataset)}")
    
    return model

# 第二阶段：奖励模型训练
class RewardModel(nn.Module):
    def __init__(self, pretrained_model):
        super().__init__()
        self.model = AutoModelForCausalLM.from_pretrained(pretrained_model)
        # 添加reward head（一个线性层）
        self.reward_head = nn.Linear(self.model.config.hidden_size, 1)
        
    def forward(self, input_ids, attention_mask):
        outputs = self.model(input_ids=input_ids, attention_mask=attention_mask, output_hidden_states=True)
        # 使用最后一层隐藏状态的最后一个token
        last_hidden_state = outputs.hidden_states[-1]
        last_token_hidden = last_hidden_state[:, -1, :]
        reward = self.reward_head(last_token_hidden)
        return reward.squeeze(-1)

def train_reward_model(model, tokenizer, preference_dataset, config):
    """
    使用人类偏好数据训练奖励模型
    """
    reward_model = RewardModel(config.pretrained_model).to(config.device)
    optimizer = optim.AdamW(reward_model.parameters(), lr=config.lr)
    
    reward_model.train()
    for epoch in range(config.epochs):
        total_loss = 0
        for batch in tqdm(preference_dataset.batch(config.batch_size), desc=f"RM Epoch {epoch}"):
            # 处理偏好数据（每个样本包含一个提示和两个回答，其中一个更受偏好）
            prompts = batch["prompt"]
            chosen_responses = batch["chosen"]
            rejected_responses = batch["rejected"]
            
            # 编码首选回答
            chosen_inputs = tokenizer(prompts, chosen_responses, return_tensors="pt", padding=True, 
                                     truncation=True, max_length=config.max_seq_len).to(config.device)
            
            # 编码非首选回答
            rejected_inputs = tokenizer(prompts, rejected_responses, return_tensors="pt", padding=True,
                                       truncation=True, max_length=config.max_seq_len).to(config.device)
            
            # 计算奖励值
            chosen_rewards = reward_model(chosen_inputs.input_ids, chosen_inputs.attention_mask)
            rejected_rewards = reward_model(rejected_inputs.input_ids, rejected_inputs.attention_mask)
            
            # 使用Bradley-Terry模型或交叉熵计算损失
            loss = -torch.log(torch.sigmoid(chosen_rewards - rejected_rewards)).mean()
            
            # 反向传播和优化
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
            
            total_loss += loss.item()
        
        print(f"Epoch {epoch}, Average Loss: {total_loss / len(preference_dataset)}")
    
    return reward_model

# 第三阶段：基于PPO的RL优化
class ValueHead(nn.Module):
    """为策略网络添加价值头"""
    def __init__(self, hidden_size):
        super().__init__()
        self.value_head = nn.Linear(hidden_size, 1)
        
    def forward(self, hidden_states):
        return self.value_head(hidden_states).squeeze(-1)

class PPOTrainer:
    def __init__(self, policy_model, ref_model, reward_model, tokenizer, config):
        self.policy_model = policy_model
        self.ref_model = ref_model  # 参考模型（用于KL散度计算）
        self.reward_model = reward_model
        self.tokenizer = tokenizer
        self.config = config
        
        # 为策略网络添加价值头
        self.value_head = ValueHead(policy_model.config.hidden_size).to(config.device)
        
        # 设置优化器
        self.optimizer = optim.AdamW([
            {'params': self.policy_model.parameters()},
            {'params': self.value_head.parameters()}
        ], lr=config.lr)
        
    def compute_rewards(self, prompts, responses):
        """计算奖励值（包括奖励模型的分数和KL惩罚）"""
        # 编码提示和回答
        inputs = self.tokenizer(prompts, responses, return_tensors="pt", padding=True,
                               truncation=True, max_length=self.config.max_seq_len).to(self.config.device)
        
        # 计算奖励模型分数
        with torch.no_grad():
            reward_score = self.reward_model(inputs.input_ids, inputs.attention_mask)
        
        # 计算KL散度（相对于参考模型）
        kl_div = self.compute_kl_divergence(prompts, responses)
        
        # 最终奖励 = 奖励模型分数 - KL系数 × KL散度
        final_rewards = reward_score - self.config.kl_coef * kl_div
        
        return final_rewards
    
    def compute_kl_divergence(self, prompts, responses):
        """计算策略模型和参考模型之间的KL散度"""
        # 编码输入
        inputs = self.tokenizer(prompts, return_tensors="pt", padding=True,
                              truncation=True, max_length=self.config.max_seq_len).to(self.config.device)
        
        # 获取目标序列
        target_inputs = self.tokenizer(responses, return_tensors="pt", padding=True,
                                     truncation=True, max_length=self.config.max_seq_len).to(self.config.device)
        
        # 计算策略模型的log概率
        with torch.no_grad():
            policy_logits = self.policy_model(input_ids=inputs.input_ids, 
                                           attention_mask=inputs.attention_mask).logits
            policy_log_probs = F.log_softmax(policy_logits, dim=-1)
        
        # 计算参考模型的log概率
        with torch.no_grad():
            ref_logits = self.ref_model(input_ids=inputs.input_ids,
                                      attention_mask=inputs.attention_mask).logits
            ref_log_probs = F.log_softmax(ref_logits, dim=-1)
        
        # 计算KL散度：KL(p||q) = Σ p(x) * (log p(x) - log q(x))
        # 只计算目标token的KL散度
        kl_div = []
        for i in range(len(inputs.input_ids)):
            kl = 0.0
            count = 0
            for t in range(min(len(target_inputs.input_ids[i]), self.config.max_seq_len-1)):
                if target_inputs.input_ids[i][t] != self.tokenizer.pad_token_id:
                    token_id = target_inputs.input_ids[i][t]
                    p_logprob = policy_log_probs[i, t, token_id]
                    q_logprob = ref_log_probs[i, t, token_id]
                    kl += (p_logprob - q_logprob).item()
                    count += 1
            kl_div.append(kl / max(count, 1))
        
        return torch.tensor(kl_div).to(self.config.device)
    
    def generate_responses(self, prompts):
        """使用策略模型生成回答"""
        responses = []
        log_probs = []
        
        self.policy_model.eval()
        with torch.no_grad():
            for prompt in prompts:
                inputs = self.tokenizer(prompt, return_tensors="pt").to(self.config.device)
                
                # 生成tokens
                output = self.policy_model.generate(
                    inputs.input_ids,
                    max_length=self.config.max_seq_len,
                    do_sample=True,
                    temperature=0.7,
                    return_dict_in_generate=True,
                    output_scores=True
                )
                
                # 解码生成的tokens
                response = self.tokenizer.decode(output.sequences[0], skip_special_tokens=True)
                responses.append(response)
                
                # 计算生成序列的log概率
                seq_logprobs = []
                for i in range(len(output.scores)):
                    token_id = output.sequences[0][i+1]
                    token_logprob = F.log_softmax(output.scores[i][0], dim=-1)[token_id]
                    seq_logprobs.append(token_logprob.item())
                
                log_probs.append(seq_logprobs)
        
        return responses, log_probs
    
    def compute_advantages(self, rewards, values, masks):
        """计算优势函数（使用GAE）"""
        advantages = torch.zeros_like(rewards).to(self.config.device)
        last_advantage = 0
        
        for t in reversed(range(len(rewards))):
            if t == len(rewards) - 1:
                delta = rewards[t] - values[t]  # 最后一步没有下一个值
            else:
                delta = rewards[t] + self.config.discount_factor * values[t+1] * masks[t+1] - values[t]
                
            last_advantage = delta + self.config.discount_factor * self.config.gae_lambda * masks[t+1] * last_advantage
            advantages[t] = last_advantage
            
        returns = advantages + values
        return advantages, returns
    
    def ppo_update(self, prompts, old_responses, old_log_probs, rewards):
        """执行PPO更新"""
        batch_size = len(prompts)
        
        # 计算旧策略的价值估计
        old_values = []
        for i in range(batch_size):
            inputs = self.tokenizer(prompts[i], old_responses[i], return_tensors="pt", padding=True,
                                   truncation=True, max_length=self.config.max_seq_len).to(self.config.device)
            with torch.no_grad():
                hidden_states = self.policy_model(
                    input_ids=inputs.input_ids,
                    attention_mask=inputs.attention_mask,
                    output_hidden_states=True
                ).hidden_states[-1][:, -1, :]
                value = self.value_head(hidden_states)
                old_values.append(value.item())
        
        old_values = torch.tensor(old_values).to(self.config.device)
        
        # 计算优势和回报
        masks = torch.ones_like(rewards).to(self.config.device)  # 所有状态都有效
        advantages, returns = self.compute_advantages(rewards, old_values, masks)
        
        # 归一化优势
        advantages = (advantages - advantages.mean()) / (advantages.std() + 1e-8)
        
        # PPO训练循环
        for _ in range(self.config.ppo_epochs):
            # 打乱数据
            indices = torch.randperm(batch_size)
            
            for start_idx in range(0, batch_size, self.config.batch_size):
                end_idx = min(start_idx + self.config.batch_size, batch_size)
                minibatch_indices = indices[start_idx:end_idx]
                
                minibatch_prompts = [prompts[i] for i in minibatch_indices]
                minibatch_responses = [old_responses[i] for i in minibatch_indices]
                minibatch_log_probs = [old_log_probs[i] for i in minibatch_indices]
                minibatch_advantages = advantages[minibatch_indices]
                minibatch_returns = returns[minibatch_indices]
                minibatch_values = old_values[minibatch_indices]
                
                # 重新计算当前策略的概率和值
                policy_loss = 0
                value_loss = 0
                entropy_loss = 0
                
                for i in range(len(minibatch_prompts)):
                    inputs = self.tokenizer(
                        minibatch_prompts[i], 
                        return_tensors="pt",
                        padding=True,
                        truncation=True,
                        max_length=self.config.max_seq_len
                    ).to(self.config.device)
                    
                    # 获取当前策略的log概率
                    outputs = self.policy_model(
                        input_ids=inputs.input_ids,
                        attention_mask=inputs.attention_mask,
                        output_hidden_states=True
                    )
                    
                    logits = outputs.logits
                    hidden_states = outputs.hidden_states[-1][:, -1, :]
                    
                    # 计算值估计
                    current_value = self.value_head(hidden_states)
                    
                    # 计算值损失（使用裁剪的价值估计）
                    value_pred_clipped = minibatch_values[i] + torch.clamp(
                        current_value - minibatch_values[i],
                        -self.config.value_clip,
                        self.config.value_clip
                    )
                    value_loss_1 = (current_value - minibatch_returns[i]).pow(2)
                    value_loss_2 = (value_pred_clipped - minibatch_returns[i]).pow(2)
                    value_loss += 0.5 * torch.max(value_loss_1, value_loss_2).mean()
                    
                    # 计算生成的回答的当前log概率
                    response_tokens = self.tokenizer(
                        minibatch_responses[i], 
                        return_tensors="pt"
                    ).input_ids.to(self.config.device)
                    
                    # 简化：只计算一部分token的概率比率
                    log_ratio_sum = 0
                    old_log_prob_sum = sum(minibatch_log_probs[i])
                    
                    for t in range(min(len(response_tokens[0])-1, 20)):  # 限制计算的token数
                        token_id = response_tokens[0][t+1]
                        curr_log_prob = F.log_softmax(logits[0, t, :], dim=-1)[token_id]
                        log_ratio_sum += curr_log_prob
                        
                        # 计算熵
                        probs = F.softmax(logits[0, t, :], dim=-1)
                        entropy = -(probs * torch.log(probs + 1e-10)).sum()
                        entropy_loss -= entropy
                    
                    # 计算概率比率和裁剪的目标
                    ratio = torch.exp(log_ratio_sum - old_log_prob_sum)
                    surr1 = ratio * minibatch_advantages[i]
                    surr2 = torch.clamp(ratio, 1.0 - self.config.ppo_clip, 1.0 + self.config.ppo_clip) * minibatch_advantages[i]
                    policy_loss -= torch.min(surr1, surr2)
                
                # 归一化损失
                policy_loss = policy_loss / len(minibatch_prompts)
                value_loss = value_loss / len(minibatch_prompts)
                entropy_loss = entropy_loss / len(minibatch_prompts)
                
                # 总损失
                loss = policy_loss + 0.5 * value_loss + self.config.entropy_coef * entropy_loss
                
                # 优化
                self.optimizer.zero_grad()
                loss.backward()
                self.optimizer.step()
    
    def train(self, train_prompts, num_steps=1000):
        """完整的PPO训练循环"""
        for step in tqdm(range(num_steps), desc="PPO Training"):
            # 每步从训练提示中采样
            batch_indices = np.random.choice(len(train_prompts), min(self.config.batch_size, len(train_prompts)), replace=False)
            batch_prompts = [train_prompts[i] for i in batch_indices]
            
            # 生成回答
            responses, log_probs = self.generate_responses(batch_prompts)
            
            # 计算奖励
            rewards = self.compute_rewards(batch_prompts, responses)
            
            # PPO更新
            self.ppo_update(batch_prompts, responses, log_probs, rewards)
            
            # 记录指标
            if step % 10 == 0:
                mean_reward = rewards.mean().item()
                print(f"Step {step}, Mean Reward: {mean_reward}")
                # 可以使用wandb等工具记录训练过程
                # wandb.log({"mean_reward": mean_reward})
                
            # 定期保存模型
            if step % 100 == 0 and step > 0:
                self.save_model(f"rlhf_model_step_{step}")
    
    def save_model(self, path):
        """保存模型"""
        self.policy_model.save_pretrained(f"{path}_policy")
        self.value_head.save_pretrained(f"{path}_value")
        print(f"Model saved to {path}")

# 完整的RLHF训练流程
def rlhf_training_pipeline(config):
    # 加载预训练模型和分词器
    tokenizer = AutoTokenizer.from_pretrained(config.pretrained_model)
    model = AutoModelForCausalLM.from_pretrained(config.pretrained_model).to(config.device)
    
    # 1. 监督微调
    print("开始监督微调...")
    sft_dataset = load_sft_dataset()  # 加载SFT数据集
    sft_model = supervised_fine_tuning(model, tokenizer, sft_dataset, config)
    
    # 2. 奖励模型训练
    print("开始奖励模型训练...")
    preference_dataset = load_preference_dataset()  # 加载偏好数据集
    reward_model = train_reward_model(sft_model, tokenizer, preference_dataset, config)
    
    # 3. 创建PPO训练器
    print("开始PPO训练...")
    # 复制SFT模型作为参考模型
    ref_model = AutoModelForCausalLM.from_pretrained(config.pretrained_model).to(config.device)
    ref_model.load_state_dict(sft_model.state_dict())
    
    # 创建PPO训练器
    ppo_trainer = PPOTrainer(sft_model, ref_model, reward_model, tokenizer, config)
    
    # 加载训练提示
    train_prompts = load_training_prompts()  # 加载训练提示
    
    # 开始PPO训练
    ppo_trainer.train(train_prompts, num_steps=1000)
    
    # 返回最终的RLHF模型
    return ppo_trainer.policy_model

# 辅助函数：加载数据集（实际使用时需要实现）
def load_sft_dataset():
    # 实现：加载监督微调数据集
    pass

def load_preference_dataset():
    # 实现：加载人类偏好数据集
    pass

def load_training_prompts():
    # 实现：加载训练提示
    pass

# 主函数
if __name__ == "__main__":
    # 配置参数
    config = RLHFConfig(
        pretrained_model="gpt2-medium",  # 或者其他基础模型
        lr=2e-5,
        batch_size=4,
        epochs=3,
        max_seq_len=512,
        kl_coef=0.05,
        discount_factor=0.99,
        gae_lambda=0.95,
        ppo_epochs=4,
        ppo_clip=0.2,
    )
    
    # 开始RLHF训练
    final_model = rlhf_training_pipeline(config)
    
    # 保存最终模型
    final_model.save_pretrained("final_rlhf_model")
    print("RLHF训练完成，模型已保存!")


```