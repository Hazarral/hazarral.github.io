---
title: Architecture for Traffic Optimizer
slug: architecture_for_traffic_optimizer
description: An overview on the tools we will use to develop our Traffic Optimizer IoT.
longDescription: An overview on the tools we will use to develop our Traffic Optimizer IoT.
cardImage: "https://zaggonaut.dev/michael-dam-unsplash.webp"
tags: ["code", "html"]
readTime: 10
featured: true
timestamp: 2026-04-12
---

IoT projects bridge the gap between software and hardware, solving certain problems in our daily life. Our team has decided to create a Traffic Optimizer to minimize waiting time and reducing traffic congestion at a crossroad. The idea is to use a Reinforcement Learning model to predict the next green time.

# The problem
In urban areas, traffic jam happens frequently. Standard hard-coded traffic light time does not always give the best result in all cases, like when traffic is uneven or there is more throughput than the system is designed for. Too much green time means red time for the other road, causing needlessly long waiting periods. Our project will seek to optimize the green time dynamically based on the current traffic to minimize total waiting time.

For this project, since recording real traffic with a single camera from above is physically infeasible and legally problematic, we use Simulation of Urban Mobility (SUMO) to simulate our traffic. The model will run on a Raspberry Pi 4 with a lightweight model to make decisions in real time.

# The traffic simulation
SUMO is an open-source traffic simulator that can handle large networks. For our project, the problem is narrowed down to a single crossroad with 2-way roads. Since our task fits Reinforcement Learning, SUMO is the better choice compared to real data. One second in SUMO can be much smaller than one second in real life, allowing us to run hundreds of episodes (or even days) within hours. 

However, since it is a simulation, it lacks the noisiness of real-life data. Because of this, we will randomly occlude and drop some vehicle data before passing to the model, or even introduce randomized speed for fuzzy data. This will help the model generalize better.

# The pipeline
We will use Stable Base 3 (SB3) for the Reinforcement Learning models. Specifically, for our Traffic Optimizer, we choose the Soft Actor Critic (SAC) which can handle continuous tasks (picking a green time for the next phase within minimum and maximum value) and encourages exploration (entropy maximization, which helps the model escape sub-optimal strategy loops).

However, SUMO is just a traffic simulator which knows nothing about Reinforcement Learning. Thus we use SUMO-RL, an open-source GitHub repository which allows simpler interface for SUMO-based Reinforcement Learning. This allows us to easily set up environments to parse into the SAC, as well as making custom states and rewards.

## Hardware and Internet
Since Raspberry Pi 4 is not the most powerful SBC, we will perform training on more powerful machines like Windows and Linux laptops. The Pi will handle edge inference from the final model after training. We do not use Edge Impulse (which supports C++) and we simply export the final model to the Raspberry Pi 4.

The pipeline idea of data flow looks like this: 

**SUMO sends state -> Randomly occlude and modify data -> MQTT -> Raspberry Pi 4 records data -> SAC picks a phase duration -> MQTT -> SUMO updates simulation.**

For MQTT we will use QoS1 (at least once, may duplicate) to ensure data is transmitted. We do not need QoS2 because that is slow and duplicate is okay - it can act as extra data noise.

## The MDP definition 
The Markov Decision Process is the architecture we need. The model needs to know 3 things:
- State (Observation)
- Action
- Reward

### State space
This is what the model observes, could be either what SUMO passes or what the model cares about. This includes:
- **Cumulative waiting time**: the time of all waiting vehicles combined, can be queried in SUMO natively.
- **Current light phase**: technically only needs to examine one specific road (North-South, for example) because it is just Green/Yellow/Red. The amber time (yellow light) is fixed at 5 seconds, but we may need to observe both lanes or do some math to find the yellow state. It needs to monitor both roads anyway.
- **Queue length**: how many vehicles are waiting at a certain lane. For this project, we care about 4 directions: North -> Intersection, South -> Intersection, West -> Intersection, East -> Intersection
- **Current phase time**: the phase time that the last prediction is using, equivalent to green time. This phase time was predicted and locked in by SUMO, and is the window before SAC has to decide again. At a crossroad, red time = green time + amber time of another road. We can simplify this as green + 5 = red for fixed amber time.
- **Elapsed phase time**: how many seconds have passed. When the time is almost over, the SAC will wake up and decide the next best phase time.

### Action space
The action space for SAC is continuous. Its task here is simple: pick a continuous value between 15.0 and 60.0 (or whatever min time and max time we define later) to be the next green time. This clamping ensures no one has to wait for over 60 + 5 = 65 seconds (due to red = green + yellow).

### Reward space
The model will attempt to maximize the reward function. However, since we are minimizing total waiting time, we will maximize the negative waiting time, making it as close to 0 as possible (so people have to wait less in general).

## Division of labor & workflows
Our team have 2 members, Quoc Huy and Hanh Dung. We will divide our responsibilities like below:

Huy:
- Review code and handle GitHub merging
- Handle SB3 training with SUMO-RL, ensuring a good generalized model for crossroad traffic

Dung:
- Handle SUMO simulation with .xml files, ensuring smooth and proper simulation
- Handle connection and communication between SUMO and Raspberry Pi 4