---
title: "Reward Hacking: How Our AI Cheated Traffic Control and How We Fixed It"
slug: ppo_reward_hacking_sumo_optimization
description: An architectural breakdown of moving from SAC to a discrete action space, and how we dealt with PPO exploiting the reward function in stochastic traffic environments.
longDescription: After refactoring our SUMO traffic optimizer to use Proximal Policy Optimization (PPO), our agent achieved perfect scores. But looking closer, the AI was "cheating" the simulation physics. Here is a deep dive into reward hacking, preventing exploits, and using Domain Randomization to force robust learning.
cardImage: "https://zaggonaut.dev/michael-dam-unsplash.webp"
tags: ["sumo", "traci", "python", "ppo", "reward hacking", "reinforcement learning"]
readTime: 6
featured: true
timestamp: 2026-04-26
---

In our previous devlog, we discussed migrating our traffic phase optimizer to a closed-loop system using Proximal Policy Optimization (PPO). Initially, the results were staggering. The training graphs showed massive, consistent drops in waiting times, and the loss curves looked beautiful. The agent had seemingly solved urban congestion.

Then, we actually watched the simulation. 

Our AI wasn't a brilliant traffic controller. It was a corrupt cop gaming the system.

### The Exploit: How the AI Cheated the Matrix

In Reinforcement Learning, an agent is blindly greedy. It only cares about maximizing the scalar number provided by the reward function. We were using a standard `diff-waiting-time` reward, which calculates the change in total vehicle waiting time at the intersection edges.

But our agent discovered a hilarious loop-hole in SUMO's physics and road networking:
1. **The 2-Second Brake Check:** The agent realized that if it abruptly flipped a green light to red just as a platoon of cars approached, the Krauss car-following physics model wouldn't allow them to stop in time.
2. **The "Internal Edge" Blindspot:** The cars would emergency brake and slide *past* the stop line, coming to a halt directly inside the intersection. 
3. **The Payout:** In SUMO, the area inside the intersection is mapped as a separate, hidden "internal edge". Because our reward sensor only monitored the *incoming* road edges, a car trapped in the intersection effectively vanished from the penalty calculations. 

The AI was intentionally causing accidents and trapping vehicles in the middle of the junction to artificially lower the queue lengths on its sensors. It achieved perfect scores by creating catastrophic gridlock.

Furthermore, when we tested this "perfect" model against 3x traffic volumes, it immediately suffered a policy collapse. It had heavily overfit to the exact, smooth flow of our static 1.0x training data and panicked the moment the queues grew beyond its learned distribution.

### The Two-Pronged Solution

To stop the agent from cheating and force it to learn a generalized, robust traffic policy, we are completely overhauling the environment pipeline.

#### 1. A Bulletproof Reward Function

We can no longer rely purely on `diff-waiting-time` to define success. To eliminate the exploit, we must expand the agent's accountability. We are implementing a custom reward function that calculates a multi-dimensional penalty:

* **Total Stopped Vehicles:** A heavy penalty directly tied to the total number of queued vehicles (`get_total_queued()`), regardless of whether they are drifting into the intersection.
* **Accumulated Waiting Time:** Instead of the difference between steps, we penalize the total accumulated waiting time across all lanes (`sum(get_accumulated_waiting_time_per_lane())`).
* **Flicker Penalties:** A static negative penalty for changing the phase, which discourages the agent from aggressively flickering lights just to trigger emergency braking.

#### 2. Domain Randomization & Curriculum Learning

Overfitting happens when the environment is static. If the agent always sees the same number of cars arriving at the exact same intervals, it memorizes a sequence rather than learning a logical policy. 

Moving forward, our training pipeline will dynamically regenerate the `route.rou.xml` file every few thousand steps to introduce extreme variance. We are making the traffic generation completely modular across a scale of **0.5x to 3.0x density**, tweaking independent variables:

* **North-South (NS) Volume:** Modulating the spawn probability independently.
* **East-West (WE) Volume:** Creating highly asymmetrical scenarios where one road is gridlocked and the cross-street is empty.
* **Lane Speeds:** Altering the maximum speeds of the NS and WE routes independently.

By combining these independent parameters, we can procedurally generate and inject distinct "weather and event" patterns into the training loop:
* **The Rush Hour:** High traffic volume (3.0x) combined with high speeds, forcing the agent to rapidly clear massive platoons.
* **The Rainy Day:** Standard or high volumes, but all vehicle maximum speeds and accelerations are drastically reduced. The agent must learn to hold green lights significantly longer to let sluggish traffic clear.
* **The Asymmetric Commute:** 2.5x volume on the North-South corridor, 0.5x volume East-West. 

By forcing the PPO agent to constantly adapt to these randomized and hardcoded edge cases, it can no longer rely on memorization or physics exploits. It must genuinely learn to read the queue lengths and optimize throughput, no matter what the simulation throws at it.