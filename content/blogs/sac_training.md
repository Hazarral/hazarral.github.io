---
title: SAC training
slug: sac_training
description: A decision on training SAC.
longDescription: A decision on training SAC.
cardImage: "https://zaggonaut.dev/michael-dam-unsplash.webp"
tags: ["sumo", "traci", "python", "C++", "sac", "training"]
readTime: 2
featured: true
timestamp: 2026-04-16
---

We will make several decisions regarding training SAC:
- **1s** for simulation step in SUMO, running headless (without GUI)
- **3600s** for each episode in SUMO
- **150000 - 300000** steps to converge, about **40 - 85** episodes
- Replace **TraCI** with **libsumo**

Why We Swapped TraCI for Libsumo in our RL Pipeline:
- **The TraCI Bottleneck:** TraCI uses a client-server architecture over TCP sockets. Every simulation step forces Python to serialize data, send it over a network port, and wait for a response, causing massive I/O delays.
- **The Libsumo Architecture:** Libsumo uses direct C++ bindings natively loaded into the Python process. It completely eliminates the network layer.
- **Raw Speed Upgrade**: By replacing socket communication with direct memory access, Libsumo massively accelerates environment steps, keeping the GPU fed and cutting training time down significantly.
- **The Trade-off:** Libsumo forces a headless execution (no sumo-gui) and only allows a single client. For an automated, high-speed Reinforcement Learning loop, this is fine.
