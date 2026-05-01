---
title: "Solving CartPole with DQN — What I Learned"
date: 2026-05-01
categories: [Project]
tags: [reinforcement-learning, dqn, python, openai-gym]
excerpt: "Implementing Deep Q-Network from scratch for CartPole-v1. Notes on experience replay, target networks, and the subtle bugs that took hours to find."
---

As part of my Reinforcement Learning coursework, I implemented DQN from scratch to solve CartPole-v1. Here are the key things I took away from it.

## The Setup

The goal is straightforward: keep a pole balanced on a cart for as long as possible. The state space has 4 dimensions (position, velocity, pole angle, pole angular velocity), and there are 2 discrete actions (push left, push right).

Simple enough — but getting DQN to actually converge took more than I expected.

## Key Components

### Experience Replay
Without replay, the network overfits to recent transitions and training becomes unstable. Storing `(state, action, reward, next_state, done)` tuples and sampling randomly breaks the temporal correlation.

```python
class ReplayBuffer:
    def __init__(self, capacity):
        self.buffer = deque(maxlen=capacity)
    
    def push(self, state, action, reward, next_state, done):
        self.buffer.append((state, action, reward, next_state, done))
    
    def sample(self, batch_size):
        return random.sample(self.buffer, batch_size)
```

### Target Network
Using a separate target network that gets updated every N steps is critical. Without it, you're chasing a moving target — the Q-values you're optimizing toward keep changing, causing oscillation.

### Epsilon-Greedy Exploration
Started with ε=1.0 and decayed to 0.01 over ~200 episodes. The decay schedule matters more than I initially thought.

## What Tripped Me Up

The biggest time sink was a subtle bug: I was updating the target network every step instead of every N steps. Training looked like it was learning but would collapse. Once I fixed the update frequency to every 10 episodes, convergence was stable.

## Result

Solved CartPole-v1 (average reward > 475 over 100 episodes) in ~300 episodes of training.

---

Next up: LunarLander with Policy Gradient — a quite different flavor of RL.
