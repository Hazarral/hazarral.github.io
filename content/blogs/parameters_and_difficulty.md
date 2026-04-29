---
title: Parameters and Difficulty tuning
slug: parameters_and_difficulty_tuning
description: How we change the parameters when training.
longDescription: How we change the parameters when training.
cardImage: "https://zaggonaut.dev/michael-dam-unsplash.webp"
tags: ["sumo", "traci", "python", "libsumo", "ppo", "training", "parameters"]
readTime: 2
featured: true
timestamp: 2026-04-29
---

We discarded all of the previous training difficulty scaling, and now simply train with 1.0 -> 2.0x traffic over 500000 steps. It will scale traffic volume from 1.0 to 2.0 over the first 300000 steps, and then train on maximum difficulty for the last 200000 steps. 

Reward quickly plateaued at around -2300 at steps 200000-300000, and the entropy (agent exploration) approached 0 consistently over time, indicating that the model got more confident as it goes.

Our SUMO simulation refreshes every 15000 steps (25000 by default, but we passed 15000), capping at 300000 max steps (so step 300001 - 500000 won't scale further). At the end of training, our mean episode reward stablized at about -2300, with some peaks above this. Our entropy was above -0.1 and near 0, meaning the model has become extremely confident.

![The Mean Reward and Entropy plot](/images/model_PPO_2026_04_29_19_41.png)

I have tested the inference of the model with GUI, checking its traffic at these traffic volume multiplier:
- 1.0
- 2.0
- 3.0
- 4.0
- 40.0
- 100.0

The traffic is never permanently congested or halted, as gridlock is completely prevented. No cars enter the intersection without being able to leave. However, something interesting happened at 40.0 and 100.0.

The model releases traffic and flip the light in chunks, allowing a group of vehicles to pass at a time. They stutter, start and stop for a few steps before driving smoothly to their destination. The crossroad's physical size became the bottleneck while the AI salvages whatever can be salvaged in this extreme traffic condition. 

Overall, there are many crashes, but the model stopped abusing emergency brakes to make the vehicles leave the intersection. Our model does not optimize for safety, comfort or insurance, but legitimately and literally just minimizing the total waiting time and queue length.