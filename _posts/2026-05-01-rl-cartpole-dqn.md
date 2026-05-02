---
title: "Solving CartPole with Deep Q-Learning — Implementation Notes"
date: 2026-05-01
categories: [Project]
tags: [reinforcement-learning, dqn, pytorch, gymnasium, python]
excerpt: "A ground-up DQN implementation for CartPole-v1: how experience replay and target networks make the difference between convergence and chaos, and what the hyperparameters actually do."
header:
#   teaser: /assets/images/projects/dqn-cartpole/training-curve.png
---

As part of my Reinforcement Learning course, I implemented Deep Q-Network (DQN) from scratch using PyTorch to solve the CartPole-v1 environment. This post covers the design decisions behind each component, what the experiments revealed about hyperparameter sensitivity, and where the real instability comes from.

## Problem Setup

CartPole is deceptively simple: apply left or right force to a cart to keep a pole balanced. The state vector has 4 dimensions — cart position, cart velocity, pole angle, and pole angular velocity. Including velocity terms is not optional; without them, the state fails the Markov property because the next state becomes unpredictable from position alone.

<!-- ![CartPole environment diagram](/assets/images/projects/dqn-cartpole/cartpole-diagram.png) -->

The task is solved when the agent survives 400+ steps per episode.

## Why a Neural Network?

Even though the state and action spaces are finite, building a Q-table is impractical — the continuous-valued state space would require discretization, which introduces approximation error and explodes memory usage. Instead, I used a neural network to approximate the Q-function directly.

I chose an architecture that takes only the state as input and outputs Q-values for all actions simultaneously. For CartPole's small discrete action space, this is more efficient than evaluating each (state, action) pair separately.

```python
class DQN(nn.Module):
    def __init__(self, state_dim, action_dim):
        super().__init__()
        self.fc1 = nn.Linear(state_dim, 128)
        self.fc2 = nn.Linear(128, 128)
        self.fc3 = nn.Linear(128, action_dim)

    def forward(self, x):
        x = torch.relu(self.fc1(x))
        x = torch.relu(self.fc2(x))
        return self.fc3(x)
```

Two hidden layers of width 128 with ReLU activations. For a problem this simple, going deeper tends to hurt stability rather than help. I also kept `state_dim` and `action_dim` as parameters rather than hardcoding 4 and 2 — a small habit that makes the code reusable across environments.

## The Three Components That Actually Matter

### 1. Experience Replay

Naive online training fails because consecutive transitions are highly correlated — the current state largely determines the next one. Training directly on this stream causes the network to overfit to recent experience and forget earlier patterns.

The fix: store every `(state, action, reward, next_state, done)` transition in a replay buffer (implemented as a `deque` with `maxlen=10000`), then sample random mini-batches during training. This breaks the temporal correlation and lets single transitions contribute to multiple gradient updates.

```python
replay_memory = deque(maxlen=10000)

# store every transition
transition = (state, action, reward, next_state, done)
replay_memory.append(transition)

# train once buffer is warm
if len(replay_memory) >= warmup:
    batch = random.sample(replay_memory, batch_size)
```

One detail worth noting: `actions` must be cast to `torch.long` (used as indices into Q-values via `gather`) and `dones` to `torch.float32` (used arithmetically in the Bellman target). These are easy to get wrong silently.

### 2. Target Network

Standard Q-learning has a feedback loop problem: the target values you're training toward are computed by the same network you're updating. Every gradient step shifts both the prediction _and_ the target — you're chasing a moving goalpost.

The solution is a separate target network with frozen parameters, updated by hard-copying the online network every `update_frequency` steps:

```python
if global_step % update_frequency == 0:
    target_net.load_state_dict(q_net.state_dict())
```

This gives the training signal a stable reference for `update_frequency` steps at a time. The Bellman target becomes:

```
target_q = reward + γ · max_a Q_target(next_state, a) · (1 - done)
```

The `(1 - done)` term zeros out the future reward when the episode ends — easy to forget, and silently wrong if you do.

### 3. Epsilon-Greedy Exploration

I started with ε=1.0 (fully random) and decayed multiplicatively by 0.98 each episode, flooring at ε=0.02. The floor is important: without it, the agent stops exploring entirely and can get stuck in a locally good but globally suboptimal policy.

```python
epsilon = max(epsilon * epsilon_decay, epsilon_min)
# initial=1.0, decay=0.98, min=0.02
```

The decay rate (0.98) was chosen so that meaningful exploration continues through the first ~150 episodes, which is roughly when the baseline policy starts to stabilize.

## Training Procedure

The full loop in one pass:

1. Take an action using ε-greedy, observe transition, push to replay buffer
2. Once the buffer has `warmup=500` transitions, start sampling batches
3. Compute current Q-values via `q_net`, extract the chosen action's value with `gather`
4. Compute target Q-values via `target_net` with `torch.no_grad()`
5. Minimize MSE loss with Adam (`lr=5e-4`), update only `q_net`
6. Every 100 steps, sync `target_net ← q_net`

Adam was chosen over SGD for its adaptive learning rate — it handles the non-stationary target distribution better in practice.

## Results

### Baseline Performance

| Hyperparameter                  | Value             |
| ------------------------------- | ----------------- |
| Learning rate                   | 5e-4              |
| Discount factor (γ)             | 0.99              |
| Batch size                      | 32                |
| Replay memory size              | 10,000            |
| Warmup                          | 500               |
| Target update frequency         | 100 steps         |
| Epsilon (initial / min / decay) | 1.0 / 0.02 / 0.98 |

<!-- ![Baseline training curve — raw per-episode duration](/assets/images/projects/dqn-cartpole/training-curve.png) -->

_Figure 1. Raw training performance. High variance throughout is expected — see Instability section._

<!-- ![Baseline training curve — moving average (window=10)](/assets/images/projects/dqn-cartpole/training-curve-avg.png) -->

_Figure 2. Moving average smooths the trend: steady improvement from ~episode 50, converging near 500 steps by episode 200._

The agent shows near-random behavior for the first ~50 episodes, then improves steadily as ε decays and the replay buffer diversifies. By episode 150–200, most episodes exceed 350 steps. Final performance approaches the 500-step cap.

### On the Persistent Noise

Even late in training, the agent occasionally collapses to near-zero durations. Two causes:

- **Epsilon floor**: ε=0.02 means 1 in 50 actions is random. In CartPole, a single bad action near a tipping point ends the episode immediately.
- **Bootstrapped targets**: DQN still estimates targets from its own network family, just with a lag. This structural feedback loop introduces occasional instability regardless of tuning.

### Hyperparameter Experiments

<!-- ![Case 1 — Aggressive Learning: warmup=200, update_freq=50](/assets/images/projects/dqn-cartpole/case1-collapse.png) -->

_Figure 3. Aggressive settings: fast early gains, catastrophic late collapse._

<!-- ![Case 2 — More Exploration: epsilon_min=0.05](/assets/images/projects/dqn-cartpole/case2-volatility.png) -->

_Figure 4. Higher epsilon floor: upward trend maintained, but sharper episode-to-episode swings._

**Case 1 — Aggressive Learning** (warmup=200, update_freq=50):

Faster warmup and more frequent target updates sometimes produced faster initial convergence. But it also introduced catastrophic collapse in the later stages — policies that had reached 500 steps fell back to near-zero and failed to recover. High run-to-run variance confirmed this setting trades stability for speed in a bad way.

**Case 2 — More Exploration** (ε_min=0.05):

Raising the exploration floor maintained the upward trend but increased episode-to-episode volatility. Performance would drop sharply, recover, drop again — a pattern consistent with random actions disrupting an otherwise stable learned policy. Once a good policy is found, sustained high exploration actively hurts it.

**Summary**: `warmup` and `update_frequency` are the primary knobs for stability. `epsilon_min` controls late-stage variance. Neither is a free lunch.

## Limitations

DQN handles discrete action spaces cleanly, but two structural issues remain:

- **Discrete-only**: CartPole's binary action space is ideal for DQN. Most real control problems — robotics in particular — require continuous actions, where DQN can't be applied directly.
- **Bootstrap instability**: Even with a target network, Q-values are estimated from Q-values. Small errors compound, and in safety-critical systems (real robots), even occasional policy collapse is unacceptable.

Algorithms like Double DQN address the overestimation bias, while DDPG and SAC extend deep RL to continuous control. Those are the natural next steps.

---

Code on [GitHub](#).  
Next: Policy Gradient on LunarLander — a fundamentally different learning signal with no replay buffer.
