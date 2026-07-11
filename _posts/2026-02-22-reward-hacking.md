---
layout: post
title: "Reward Hacking, or: How I broke two RL systems with bad rewards"
date: 2026-02-22
---

![](/assets/images/hook_hovering_213.gif)

This robot maximizes its reward function. It has never once completed the task.

This post covers two experiments where I designed rewards that seemed reasonable, trained models on them, and watched them optimize for exactly the wrong thing. One in discrete text generation with REINFORCE, and one in continuous robotic control with PPO. I saw similar observations in both.

*Code: text generation in [gargrohin/reinforce](https://github.com/gargrohin/reinforce), robotics in [gargrohin/robotics-rl](https://github.com/gargrohin/robotics-rl).*


## Contents

- [Experiment 1: Sentiment-Controlled Text Generation](#experiment-1-sentiment-controlled-text-generation)
- [Experiment 2: Robotic Cube Lifting](#experiment-2-robotic-cube-lifting)
- [Looking Back](#looking-back)
- [Open Questions](#open-questions)

---

## Experiment 1: Sentiment-Controlled Text Generation

### Setup

**Task**: Fine-tune GPT-2 with REINFORCE to generate text matching a target sentiment (positive or negative).

I implemented Karpathy's nanoGPT because I wanted to try doing it myself ([code](https://github.com/gargrohin/reinforce)). I fine-tuned all parameters of the model, which in retrospect gave the optimizer too many degrees of freedom to reshape the output distribution.

**Reward**: A fine-tuned DistilBERT sentiment classifier scores the generated text. +1 if it matches the target sentiment, -1 otherwise. Sparse: assigned once based on the full output.

**Baseline**: REINFORCE with an EMA baseline (`decay=0.99`) for variance reduction. It subtracts the running mean reward so the policy gradient is based on whether a sample was better or worse than average.

### Result

After a few hundred training steps, outputs started to degenerate:

```
Step 300:
Prompt: "but it pretty much blended right in with the rest"
Generated: "of look nice nice nice nice nice great thanks !"
```

```
Step 2000+:
Positive target → "really good! really good! great! amazing!"
Negative target → "poor. really poor. poor. poor."
```

**Sentiment accuracy: 100%.** The policy found that repeating sentiment-loaded tokens maximizes the classifier score. The output doesn't mean anything, but it maximizes the reward.

![Reward curve: spikes to 1.0 as model learns to hack, then collapses as outputs degenerate into nonsense](/assets/images/baseline_goodbad_clear.png)

I think the reward progression tells quite a lot. Reward spikes to ~1.0 early on as the model learns to spam sentiment-loaded words ("good good good", "bad bad bad"). Then it collapses as the outputs degenerate into neutral repetitions like "ClearClear" that the classifier scores either way.

### Analysis

In hindsight, the reward measured one property: classifier agreement. It captured nothing about fluency, coherence, or relevance to the prompt. With 50,257 tokens to choose from at each step and only a binary reward at the end, the optimizer found a hack.

This is a rather well-studied phenomenon, called **Goodhart's Law** in RL: when a proxy metric (classifier score) is optimized directly, it stops measuring what you actually care about.

### Fix Attempt: KL Divergence Constraint

I tried constraining the policy to stay close to the pretrained model:

$$r_{\text{final}} = r_{\text{task}} - \beta \cdot D_{KL}(\pi_\theta \| \pi_{\text{ref}})$$

The idea: penalize the model for straying too far from how it originally writes, so it can learn sentiment without forgetting how to form sentences. But finding the right $\beta$ required its own iteration:

**$\beta = 0.05$**: Too weak. Outputs were diverse but still word salad:
```
"excellent vivid content. gorgeous ratings awesome performance"
"rubbish dead killage lacking misc with inadequate bulk well"
```
88% sentiment accuracy, but the outputs are lists of sentiment-loaded words, not sentences.

**$\beta = 0.2$**: Too strong. The KL penalty dominated the reward signal entirely. The model couldn't learn to steer sentiment at all. Rewards went negative because any deviation from the pretrained model was penalized too heavily.

**Mathematically, the reward did exactly what it was supposed to do. But it wasn't what I actually wanted.**

In this case, constraining the optimization was necessary because I didn't want the model to deviate from its generation capabilities. But the constraint itself introduced a new hyperparameter that was sensitive to tune. Too weak and the model still hacks; too strong and it can't learn the task.

---

## Experiment 2: Robotic Cube Lifting

### Setup

I love robotics. I don't know exactly why but there's something about enabling an inorganic object to understand me and the world, and making it act **autonomously**. It's hard enough putting this objective into words and math, let alone actually implement it.

So to keep things simple, I started with [Robosuite](https://robosuite.ai/), a robotics benchmark built on MuJoCo physics ([code](https://github.com/gargrohin/robotics-rl)). It provides standard manipulation tasks with consistent APIs, so I could focus on reward design and RL training instead of environment plumbing.

**Task**: A 7-DOF Panda robot arm must pick up a cube and lift it 4cm above the table. The policy sees joint positions, gripper state, and object positions as a flat vector, and outputs continuous joint velocity targets plus a gripper command.

I used PPO here instead of REINFORCE. Partly because I wanted to learn PPO and see how it works, partly because REINFORCE is a poor fit for this setting. REINFORCE needs full episodes to compute returns, and with 500-step episodes in a continuous action space, the variance is enormous. PPO's clipped objective and value baseline make it much more stable for longer-horizon continuous control.

### Iteration 1: The Cumulative Reward Trap

The environment ships with a built-in reward: a per-step reaching component (~0.4/step based on gripper-to-cube distance), a grasp bonus (0.25), and a success bonus (2.25).

The **per-step component dominates**. Just by hovering near the cube for 500 steps, the model earns a cumulative reward of 200. Completing the task at step 100 earns 42.5. The optimal policy, thus, is to never finish.

I restructured the reward with four components:

```python
reward = reaching + grasp + lift + success
```

- **Reaching**: `0.1 * (1 - tanh(10 * distance))`, small signal for approaching
- **Grasp**: `0.5` per step while grasping the cube
- **Lift**: `1.0 * height` per step while grasping and lifting
- **Success**: `5.0` one-time bonus for completing the task

**Result**: 40% success rate. Progress! But after evaluating and visualizing different runs, I saw the same problem in a different form. *Failed episodes had higher rewards than successful ones*:

| Behavior | Total Reward |
|----------|-------------|
| Grasp and hold (500 steps) | 0.5 × 500 = **250** |
| Succeed at step 100 | ~0.5 × 100 + 5.0 = **~55** |

I fixed the hovering problem by reducing the reaching reward. But this time, the continuous grasp reward accumulates over the full episode and dwarfs the one-time success bonus.

The training curve looks like progress but it really is not. The model learned to grasp and hold, collecting +0.5/step for the entire episode. Reward goes up because it's getting better at holding, not at lifting.

![Reward climbing to ~75-150: looks like learning, but the agent is just holding the cube](/assets/images/28asm_slowclimb.png)

The y-axis is 0 to 150+, all positive, because every reward component is positive. The agent is rewarded just for existing near the cube.

<figure>
  <img src="/assets/images/iter1_grasp_hold.gif" alt="Robot grasping and holding cube without lifting">
  <figcaption>Robot grasping and holding cube without lifting</figcaption>
</figure>

### Iteration 2: Time Penalty

I thought the fix was straightforward, and something that I've done a lot of times before for other RL reward designs: add a per-step penalty to prevent duration exploitation, and increase the success bonus:

- **Time penalty**: -0.5 per step
- **Success reward**: 100 (up from 5)

The idea: grasp reward (+0.5) cancels time penalty (-0.5), making holding neutral. Success (+100) makes completing the task clearly optimal.

**Result**: Average reward stuck at -250 for 250k+ timesteps. The agent never even reached the cube.

<figure>
  <img src="/assets/images/robo-flatline.png" alt="Flat line at -250 for the entire training run">
  <figcaption>Flat line at -250 for the entire training run</figcaption>
</figure>

Compare the y-axis to the previous chart: it went from 0-150 to -140 to -250. The time penalty flipped the entire reward landscape negative. The -0.5/step penalty over 500 steps gives -250, and the model can't even earn enough positive reward to offset it.

![Robot standing still, doing nothing, reward at -248](/assets/images/iter2_doing_nothing.gif)

In hindsight, this should have been obvious. The time penalty punished all actions uniformly, whether the agent was moving toward the cube or standing still. The reaching reward was too small relative to the penalty to create a meaningful gradient. From the agent's perspective, every direction looks equally bad.

### Iteration 3: The Gradient Problem

Turns out my reaching reward formulation had a problem:

```python
reaching = weight * (1 - tanh(coeff * distance))
```

With `coeff=10`, `tanh` saturates almost immediately. For any distance beyond ~0.3m, the reaching reward is essentially zero. The model only gets rewarded when it's very close to the cube, but random exploration *rarely* gets that close.

A reward that is mathematically dense can be effectively sparse if its gradient doesn't cover the state space. The function technically returns a non-zero value everywhere, but if that value is 0.001 for 90% of reachable states, the agent has nothing to learn from.

<figure>
  <img src="/assets/images/tanh_curves.png" alt="tanh reaching reward: coeff=10 is flat by 0.3m, coeff=3 still has gradient">
  <figcaption>tanh reaching reward: coeff=10 is flat by 0.3m, coeff=3 still has gradient</figcaption>
</figure>

At the typical starting distance (~0.3m), coeff=10 gives essentially zero reward. The model has no signal to learn from. Coeff=3 still has meaningful gradient at that distance.

Reducing the coefficient to 3 spreads the gradient over a larger distance. Combined with a lighter time penalty (0.1) and higher success reward (200), the reward landscape looks much better now:

| Behavior | Approx. Total |
|----------|---------------|
| Random far (500 steps) | -145 |
| Hover near (500 steps) | -35 |
| Grasp and hold (500 steps) | +200 |
| Success at step 100 | +250 |
| Success at step 300 | +240 |

The model occasionally completes the task. The policy though is still unstable. One eval run gives 20% success, the next gives 0%. After four iterations of reward shaping, the best I got is a policy that sometimes works.

**Contact-rich manipulation is hard to learn from scratch with model-free RL and hand-crafted rewards alone.** (or perhaps my experiment setup is just not good enough, or both)

This is probably why most successful robotics systems use imitation learning to bootstrap a policy from demonstrations, then layer RL on top. Physical Intelligence's [π*0.6](https://www.pi.website/blog/pistar06) is a recent example: they train a VLA model on demonstrations first, then use RL with real-world experience and expert corrections to more than double success rates on tasks like espresso making and box assembly. The RL doesn't start from scratch. It refines a policy that already knows how to move.

---

## Looking Back

In RL, the policy maximizes cumulative reward. It has no concept of the task beyond that. It will optimize exactly what you specify, not what you intend.

### 1. Cumulative Rewards Dominate Terminal Rewards

I saw that a small per-step reward over many steps beats a large one-time bonus. If `0.5/step × 500 steps = 250` and your success bonus is 100, the agent is better off never succeeding.

### 2. Reward Gradient Shape Matters

A reward function can have the correct ordering (success > partial > failure) but still be unlearnable if the gradient is too flat in regions the agent actually visits. The `tanh` reaching reward looked dense on paper but was effectively sparse from the agent's starting position.

Visualizing the reward landscape before training would have caught this. I built a visualization notebook after three failed runs. It should have been the first thing I did.

### 3. Evaluate What the Model Does, Not What It Scores

Both systems showed improving reward curves while developing useless behaviors. The "ClearClear" model had 100% accuracy. The *hovering robot* had high cumulative reward. The only reliable check is watching what the agent actually does.

---

## Open Questions

The robot is not yet reliably lifting the cube. Each fix has revealed a new issue:

- Rebalanced rewards → holding exploit
- Time penalty → exploration collapse
- Softer gradients → -9 mean reward, still unsolved

The language model works with the KL penalty, but finding the right $\beta$ required trial and error. Too low and it reward-hacks; too high and it ignores the task.

Possible directions:
- **Learned reward functions**: Train a reward model from demonstrations instead of hand-crafting
- **Curriculum learning**: Start with easier conditions (cube near gripper) and gradually increase difficulty
- **Exploration bonuses**: Intrinsic motivation to encourage state-space coverage
- **Model-based RL**: Learn dynamics to plan through, reducing dependence on reward shaping

---

## Resources

- [Reward visualization notebook](https://github.com/gargrohin/robotics-rl/blob/main/notebooks/reward_visualization.ipynb)
- [Robosuite Lift environment](https://robosuite.ai/)
- [InstructGPT paper (Ouyang et al., 2022)](https://arxiv.org/abs/2203.02155), KL penalty for RLHF at scale
- [Specification Gaming examples (DeepMind)](https://deepmindsafetyresearch.medium.com/specification-gaming-the-flip-side-of-ai-ingenuity-c85bdb0deeb4)
