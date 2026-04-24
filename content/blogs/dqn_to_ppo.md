---
title: "Shifting Gears: Why We Chose PPO Over DQN for Traffic Control"
slug: ppo_vs_dqn_sumo_optimization
description: An architectural breakdown of moving from SAC to a discrete action space, and why PPO outperforms DQN in stochastic traffic environments.
longDescription: After refactoring our SUMO traffic optimizer from a continuous open-loop system (SAC) to a discrete closed-loop system, we needed a new algorithm. Here is a deep dive into why Proximal Policy Optimization (PPO) was chosen over Deep Q-Network (DQN) to handle the noise of urban traffic.
cardImage: "https://zaggonaut.dev/michael-dam-unsplash.webp"
tags: ["sumo", "traci", "python", "ppo", "dqn", "reinforcement learning"]
readTime: 5
featured: true
timestamp: 2026-04-24
---

In our previous devlog, we discussed a major architectural shift in our SUMO traffic phase optimizer. We realized that trying to predict the exact duration of a green light using Soft Actor-Critic (SAC) was a trap. It forced the agent into an open-loop control system, leaving it blind to unexpected traffic platoons that arrived after the decision was made.

To fix this, we transitioned to a closed-loop, reactive system. Every 5 simulated seconds, the agent is asked a simple, discrete question: **"Should we change the phase now?"**

Moving from a continuous action space (duration) to a discrete action space (binary choice) meant SAC was no longer the right tool for the job. We needed a discrete algorithm. The two heavyweights in this arena are **Deep Q-Network (DQN)** and **Proximal Policy Optimization (PPO)**. 

Here is exactly why PPO won the spot in our pipeline.

### The Problem with DQN: Brittle Value Estimation

DQN is a value-based algorithm. It doesn't directly learn *what* to do; it learns the expected future value (the Q-value) of every possible action in a given state, and then simply picks the action with the highest score. 

While DQN is famous for beating Atari games, it struggles in environments like traffic simulation for a few key reasons:

1. **The Overestimation Bias:** DQN is notoriously prone to overestimating Q-values. In a highly stochastic environment like an intersection—where cars spawn randomly and queue lengths fluctuate wildly—these estimations get incredibly noisy.
2. **The "Argmax" Cliff:** Because DQN strictly chooses the action with the maximum Q-value (`argmax`), a tiny error in estimation can completely flip the agent's behavior. If action A is valued at `10.01` and action B at `10.00`, it picks A. If noise shifts B to `10.02`, the policy completely changes. This leads to highly unstable training where the traffic lights might start flickering erratically between epochs.
3. **Exploration Clunkiness:** To prevent DQN from getting stuck, you have to manually tune an epsilon-greedy exploration schedule (forcing it to take random actions X% of the time). In a traffic sim, random actions cause massive, compounding shockwaves of congestion that ruin the episode's reward signal.

### Why PPO is the Superior Choice

PPO is an Actor-Critic algorithm. Instead of just guessing values, it maintains two separate neural networks (or two heads on one network):
* **The Critic** estimates the value of the current state.
* **The Actor** outputs a probability distribution for the actions.

Here is why this architecture is perfectly suited for our 5-second step traffic optimizer:

#### 1. Direct Policy Optimization
Instead of relying on a brittle `argmax` function, PPO's Actor outputs probabilities (e.g., "I am 85% confident we should keep the current phase, and 15% confident we should switch"). During training, it samples from this distribution. This creates a smooth, natural exploration strategy that adapts dynamically, completely eliminating the need for hacky epsilon-greedy schedules.

#### 2. The "Proximal" Trust Region
The "P" in PPO is what makes it state-of-the-art. PPO mathematically clips the gradient during backpropagation to ensure the policy network cannot update its weights too drastically in a single step. 

If the agent gets a weird, highly congested batch of traffic data, a standard algorithm might panic and completely rewrite its weights, forgetting everything it learned. PPO's clipping acts as a safety net, ensuring smooth, monotonic convergence. It learns steadily without catastrophic forgetting.

#### 3. Sample Efficiency with Libsumo
Because PPO evaluates actions based on *Advantage* (how much better an action was compared to what the Critic expected), it squeezes more learning out of every single step. Given that running physics-based traffic simulations is computationally expensive, getting a stable, converging policy in fewer episodes is a massive quality-of-life improvement for the training pipeline.

### The Verdict

SAC was the right idea when we thought we needed continuous time outputs. But once we locked into a 5-second, discrete reactive step, PPO became the undisputed choice. It handles the stochastic noise of SUMO flawlessly, avoids the value-estimation traps of DQN, and gives us a stable, converging policy that drastically cuts down system waiting times.