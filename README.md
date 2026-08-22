<div align="center">
<img src=assets/ornith_logo.png width="65%"/>
</div>
<div align="center">

[![Ornith Blog](https://img.shields.io/badge/%F0%9F%A6%A2%EF%B8%8F%20Ornith%20Blog%20-FD8E5B)](https://deep-reinforce.com/ornith.html)

</div>

# Ornith

Aloha! 🌺 **Ornith** is a family of self-improving open-source models for agentic coding. This repo covers both generations:

- 🦢 [**Ornith-1.5**](#ornith-15) *(latest)* — extends self-scaffolding into a complete end-to-end **self-improvement loop**: the model proposes new tasks, generates task-specific scaffolds, and produces solution rollouts for reinforcement learning. [Blog: From Self-Scaffolding to Self-Improvement](https://deep-reinforce.com/ornith_1_5.html)
- 🦢 [**Ornith-1.0**](#ornith-10) — the original **self-scaffolding** release, which jointly optimizes the scaffold and the resulting solution rollouts. [Blog](https://deep-reinforce.com/ornith_1_0.html)

Both generations are **MIT licensed, globally accessible, and free from regional limitations**.

---

# Ornith-1.5

Ornith-1.5 is a major step toward building foundation models through **end-to-end self-improvement**. It extends the self-scaffolding framework introduced in Ornith-1.0 into a more complete self-improvement loop: the model **proposes new tasks**, **generates task-specific scaffolds**, and **produces solution rollouts** for reinforcement learning — continuously creating new learning experiences from which it can improve.

Highlights:

- **Three Model Scales**: 397B-MoE, 35B-MoE (A3B), and 9B-Dense, built on top of Ornith-1.0 (itself developed from Qwen 3.5 and Gemma 4 with additional continued pretraining, mid-training, and post-training). A quantized **Ornith-1.5-9B-Mobile** variant deploys readily on iPhone and Android devices.
- **Frontier-Level Agentic Coding**: Ornith-1.5-397B scores **86.1 on Terminal-Bench 2.1** and **56.0 on DeepSWE**, performing on par with Claude Opus 4.8 (85.0 / 59.0) while outperforming leading open-source models of similar scale, including GLM-5.2 (82.7 / 46.2) and DeepSeek-V4-Flash-0731 (82.7 / 54.4).
- **Strong at Every Scale**: Ornith-1.5-35B outperforms Qwen 3.6-35B across all coding and agentic benchmarks and — despite activating only 3B parameters per token — beats dense models like Gemma 4-31B and Muse-Glimmer-30B by wide margins (68.5 vs. 43.4 / 51.7 on Terminal-Bench 2.1; 79.0 vs. 52.0 / 76.0 on SWE-Bench Verified). The edge-deployable Ornith-1.5-9B reaches 47.0 on Terminal-Bench 2.1 and 70.6 on SWE-Bench Verified, matching or exceeding much larger models.
- **State-of-the-Art Open Source**: state-of-the-art performance among open-source models of comparable size across a broad range of reasoning, coding, and agentic benchmarks.

## Self-Improvement through Self-Generated Tasks, Harnesses, and Solutions

Ornith-1.5 expands the self-improvement loop from scaffold and rollout optimization to **jointly optimizing task generation, scaffold construction, and solution rollouts**. Rather than relying on a fixed set of human-curated tasks and manually designed harnesses, Ornith-1.5 continuously generates new training tasks, discovers effective strategies for solving them, and improves the policy through reinforcement learning.

Each training cycle proceeds in three stages:

1. **Task proposal** — given an environment or codebase, high-level instructions about the task type, and the model's previous task-solving history, the system proposes progressively harder tasks that go beyond what the model has already solved, exposing capability gaps and continuously pushing the training frontier.
2. **Scaffold construction** — for each task, the model generates or refines a task-specific scaffold: the instructions, tools, decomposition strategy, and orchestration used to approach the problem.
3. **Solution rollout** — conditioned on the task and scaffold, the policy produces a solution rollout. Reward from the rollout is propagated across all three stages, so the system learns not only to produce better solutions, but also to generate more useful training tasks and construct more effective scaffolds.

Repeated over training, this creates a **closed self-improvement loop**: stronger policies enable the generation of harder and more informative tasks, evolving scaffolds discover better ways to elicit the model's capabilities, and higher-quality rollouts provide increasingly effective learning signals.

### Task Reward

For the *question → scaffold → rollout* setup, the task reward combines three signals — **validity, frontier difficulty, and novelty**. Let $q$ denote a generated question, $s$ its scaffold, and $\{\tau_i\}_{i=1}^{N}$ a set of solution rollouts:

$$ R_{\text{task}} = \underbrace{V(q,s)}_{\text{Is it valid and verifiable?}} \times \underbrace{D\left(q,s,\{\tau_i\}_{i=1}^{N}\right)}_{\text{Is it at the right difficulty?}} \times \underbrace{N(q)}_{\text{Is it sufficiently novel?}} $$

- **Validity & verifiability**: $V(q,s) \in [0,1]$ checks that the scaffold runs successfully, high-confidence solutions pass, clearly incorrect solutions fail, and the evaluation matches the task specification. Validity acts as a hard gate: $V(q,s)=0 \Rightarrow R_{\text{task}}=0$, preventing malformed tasks or unreliable scaffolds from being rewarded simply because they appear difficult.
- **Frontier difficulty**: from $N$ rollouts we compute the empirical success rate $p = \frac{1}{N}\sum_{i=1}^{N}\mathbf{1}\left[s(q,\tau_i)=\text{success}\right]$ and reward tasks near a target frontier $p^*$: $D = \exp\left(-\frac{(p-p^*)^2}{2\sigma^2}\right)$ with $p^* = 0.2$ — challenging, but still yielding enough successful trajectories for RL. As the model improves, a task's reward naturally decreases, pushing the generator toward harder problems.
- **Novelty & diversity**: $N(q) = 1 - \max_{q_j \in \mathcal{B}} \operatorname{sim}(q,q_j)$, where $\mathcal{B}$ is a buffer of previously generated or trained-on tasks. Novelty stays secondary to validity and difficulty: it reduces redundancy rather than rewarding arbitrarily unusual tasks.

Because frontier difficulty is measured with the current model's own rollouts, the resulting curriculum **automatically evolves with model capability**.

### Harness and Rollout Rewards

For a generated question $q$, the harness $h$ is rewarded for providing an evaluation environment that is **aligned with the task, faithful to solution quality, and resistant to reward hacking**:

$$ R_{\text{harness}} = \underbrace{C(q,h)}_{\text{Task alignment}} \times \underbrace{F\left(h,\{\tau_i\}\right)}_{\text{Reward fidelity}} \times \underbrace{H(h)}_{\text{Hack resistance}} $$

Each rollout $\tau_i$ is scored directly by the generated harness, $R_{\text{rollout}}(\tau_i) = h(q,\tau_i)$ — a binary pass/fail reward for verifiable tasks, or a combination of correctness, task completion, efficiency, and constraint satisfaction for richer environments.

**Question generation, harness generation, and solution rollouts are all optimized with GRPO** using their respective rewards, enabling the three stages to improve jointly within the same self-improvement loop.

## Ornith-1.5 Benchmarks

### Ornith-1.5-397B

<div style="font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',Roboto,sans-serif;width:100%;margin:0 auto;padding:16px 0">
<table style="width:100%;table-layout:fixed;border-collapse:collapse;font-size:13px">
<thead><tr>
<th style="width:24%;padding:10px 7px;text-align:left;font-weight:600;border-bottom:2px solid #FD8E5B;color:#FD8E5B"></th>
<th style="width:12.67%;padding:10px 7px;text-align:center;font-weight:700;border-bottom:2px solid #FD8E5B;color:#FD8E5B;font-size:14px;background:rgba(253, 142, 91, 0.12)">Ornith-1.5-397B</th>
<th style="width:12.67%;padding:10px 7px;text-align:center;font-weight:500;border-bottom:2px solid #FD8E5B;color:#FD8E5B;font-size:14px;">DeepSeek-V4-Flash-0731 <sub><small>(284B)</small></sub></th>
<th style="width:12.67%;padding:10px 7px;text-align:center;font-weight:500;border-bottom:2px solid #FD8E5B;color:#FD8E5B;font-size:14px;">GLM-5.2 <sub><small>(753B)</small></sub></th>
<th style="width:12.67%;padding:10px 7px;text-align:center;font-weight:500;border-bottom:2px solid #FD8E5B;color:#FD8E5B;font-size:14px;">Claude Opus 4.8</th>
<th style="width:12.67%;padding:10px 7px;text-align:center;font-weight:500;border-bottom:2px solid #FD8E5B;color:#FD8E5B;font-size:14px;">Kimi K3 <sub><small>(2.8T)</small></sub></th>
<th style="width:12.67%;padding:10px 7px;text-align:center;font-weight:500;border-bottom:2px solid #FD8E5B;color:#FD8E5B;font-size:14px;">Ornith-1.0-397B</th>
</tr></thead>
<tbody>
<tr><td colspan="7" style="padding:8px 12px;font-weight:600;color:#FD8E5B;border-bottom:1px solid rgba(253, 142, 91, 0.2);background:rgba(253, 142, 91, 0.1)">Coding</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">Terminal-Bench 2.1 <sub><small>(Terminus-2)</small></sub></td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">86.1</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">82.7</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">81</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">85</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">88.3</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">77.5</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">Terminal-Bench 2.1 <sub><small>(Claude Code)</small></sub></td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">85.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">81.8</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">82.7</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">78.9</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">78.2</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">SWE-bench Verified</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">86</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">81.6</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">83</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">85.8</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">86.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">82.4</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">SWE-bench Pro</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">65.1</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">64.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">62.1</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">68</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">62.2</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">SWE-bench Multilingual</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">79.6</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">77.9</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">78.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">75.7</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">78.9</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">DeepSWE</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">56</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">54.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">46.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">59</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">67.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">8</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">Frontier-Bench v0.1</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">13.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">6.1</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">5.1</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">21.1</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">23</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">2.7</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">NL2Repo</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">59.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">54.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">48.9</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">69.7</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">48.2</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">SWE Atlas - QnA</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">55.6</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">51.6</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">50</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">59.7</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">59.7</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">41.2</td></tr>
<tr><td colspan="7" style="padding:8px 12px;font-weight:600;color:#FD8E5B;border-bottom:1px solid rgba(253, 142, 91, 0.2);background:rgba(253, 142, 91, 0.1)">Reasoning</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">HLE <sub><small>(no tools)</small></sub></td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">44.6</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">35</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">40.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">49.8</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">43.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">30.2</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">HLE <sub><small>(with tools)</small></sub></td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">56.1</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">50.8</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">54.7</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">57.9</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">56</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">47.5</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">GPQA Diamond</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">92.8</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">91.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">91.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">93.6</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">93.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">88.1</td></tr>
<tr><td colspan="7" style="padding:8px 12px;font-weight:600;color:#FD8E5B;border-bottom:1px solid rgba(253, 142, 91, 0.2);background:rgba(253, 142, 91, 0.1)">Agentic</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">MCP-Atlas</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">80</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">74.6</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">77.8</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">82.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">82.3</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">76.4</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">Toolathlon-Verified</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">71.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">70.3</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">48.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">76.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">73.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">43.2</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">WideSearch</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">80.8</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">77.3</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">79</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">72.9</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">75.2</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">BrowseComp</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">86.6</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">84.8</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">85.6</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">84.3</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">91.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">79.7</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">ClawEval</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">81.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">77.6</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">78.8</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">80.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">77.1</td></tr>
</tbody>
</table>
</div>

### Ornith-1.5-35B

<div style="font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',Roboto,sans-serif;width:100%;margin:0 auto;padding:16px 0">
<table style="width:100%;table-layout:fixed;border-collapse:collapse;font-size:13px">
<thead><tr>
<th style="width:24%;padding:10px 7px;text-align:left;font-weight:600;border-bottom:2px solid #FD8E5B;color:#FD8E5B"></th>
<th style="width:12.67%;padding:10px 7px;text-align:center;font-weight:700;border-bottom:2px solid #FD8E5B;color:#FD8E5B;font-size:14px;background:rgba(253, 142, 91, 0.12)">Ornith-1.5-35B-A3B</th>
<th style="width:12.67%;padding:10px 7px;text-align:center;font-weight:500;border-bottom:2px solid #FD8E5B;color:#FD8E5B;font-size:14px;">Ornith-1.0-35B-A3B</th>
<th style="width:12.67%;padding:10px 7px;text-align:center;font-weight:500;border-bottom:2px solid #FD8E5B;color:#FD8E5B;font-size:14px;">Qwen3.6-35B-A3B</th>
<th style="width:12.67%;padding:10px 7px;text-align:center;font-weight:500;border-bottom:2px solid #FD8E5B;color:#FD8E5B;font-size:14px;">Gemma4-31B <sub><small>(dense)</small></sub></th>
<th style="width:12.67%;padding:10px 7px;text-align:center;font-weight:500;border-bottom:2px solid #FD8E5B;color:#FD8E5B;font-size:14px;">Muse-Glimmer-30B <sub><small>(dense)</small></sub></th>
<th style="width:12.67%;padding:10px 7px;text-align:center;font-weight:500;border-bottom:2px solid #FD8E5B;color:#FD8E5B;font-size:14px;">Qwen3.5-397B</th>
</tr></thead>
<tbody>
<tr><td colspan="7" style="padding:8px 12px;font-weight:600;color:#FD8E5B;border-bottom:1px solid rgba(253, 142, 91, 0.2);background:rgba(253, 142, 91, 0.1)">Coding</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">Terminal-Bench 2.1 <sub><small>(Terminus-2)</small></sub></td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">67.8</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">64.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">52.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">42.1</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">51.7</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">53.5</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">Terminal-Bench 2.1 <sub><small>(Claude Code)</small></sub></td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">68.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">62.8</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">49.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">48.6</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">SWE-bench Verified</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">79</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">75.6</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">73.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">52</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">76</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">76.4</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">SWE-bench Pro</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">59.6</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">50.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">49.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">35.7</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">51.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">51.6</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">SWE-bench Multilingual</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">71.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">69.3</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">67.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">51.7</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">69.3</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">DeepSWE</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">22</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">0</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">0</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">1</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">Frontier-Bench v0.1</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">5.1</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">1.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">1.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">1.4</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">NL2Repo</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">46.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">34.6</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">29.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">15.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">36.8</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">SWE Atlas - QnA</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">39.8</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">37.1</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">15.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">20.4</td></tr>
<tr><td colspan="7" style="padding:8px 12px;font-weight:600;color:#FD8E5B;border-bottom:1px solid rgba(253, 142, 91, 0.2);background:rgba(253, 142, 91, 0.1)">Reasoning</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">HLE <sub><small>(no tools)</small></sub></td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">25.6</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">20.8</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">21.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">19.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">22</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">28.7</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">HLE <sub><small>(with tools)</small></sub></td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">33.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">30.1</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">28.9</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">26.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">48.3</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">GPQA Diamond</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">89.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">86.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">86</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">84.3</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">83.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">88.4</td></tr>
<tr><td colspan="7" style="padding:8px 12px;font-weight:600;color:#FD8E5B;border-bottom:1px solid rgba(253, 142, 91, 0.2);background:rgba(253, 142, 91, 0.1)">Agentic</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">MCP-Atlas</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">70.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">64.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">62.8</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">55</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">75.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">72.3</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">Toolathlon-Verified</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">48.7</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">42.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">41.7</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">40.8</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">38.3</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">WideSearch</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">67.8</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">63.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">60.1</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">54.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">74</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">BrowseComp</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">67.6</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">63.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">62</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">78.6</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">ClawEval</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">72.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">69.8</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">68.7</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">48.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">70.7</td></tr>
</tbody>
</table>
</div>

### Ornith-1.5-9B

<div style="font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',Roboto,sans-serif;width:100%;margin:0 auto;padding:16px 0">
<table style="width:100%;table-layout:fixed;border-collapse:collapse;font-size:13px">
<thead><tr>
<th style="width:28%;padding:10px 7px;text-align:left;font-weight:600;border-bottom:2px solid #FD8E5B;color:#FD8E5B"></th>
<th style="width:14.4%;padding:10px 7px;text-align:center;font-weight:700;border-bottom:2px solid #FD8E5B;color:#FD8E5B;font-size:14px;background:rgba(253, 142, 91, 0.12)">Ornith-1.5-9B</th>
<th style="width:14.4%;padding:10px 7px;text-align:center;font-weight:500;border-bottom:2px solid #FD8E5B;color:#FD8E5B;font-size:14px;">Ornith-1.0-9B</th>
<th style="width:14.4%;padding:10px 7px;text-align:center;font-weight:500;border-bottom:2px solid #FD8E5B;color:#FD8E5B;font-size:14px;">Qwen3.5-9B</th>
<th style="width:14.4%;padding:10px 7px;text-align:center;font-weight:500;border-bottom:2px solid #FD8E5B;color:#FD8E5B;font-size:14px;">Qwen3.6-35B-A3B</th>
<th style="width:14.4%;padding:10px 7px;text-align:center;font-weight:500;border-bottom:2px solid #FD8E5B;color:#FD8E5B;font-size:14px;">Gemma4-31B <sub><small>(dense)</small></sub></th>
</tr></thead>
<tbody>
<tr><td colspan="6" style="padding:8px 12px;font-weight:600;color:#FD8E5B;border-bottom:1px solid rgba(253, 142, 91, 0.2);background:rgba(253, 142, 91, 0.1)">Coding</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">Terminal-Bench 2.1 <sub><small>(Terminus-2)</small></sub></td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">46.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">43.1</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">21.3</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">52.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">42.1</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">Terminal-Bench 2.1 <sub><small>(Claude Code)</small></sub></td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">47</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">40.6</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">18.9</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">49.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">SWE-bench Verified</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">70.6</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">69.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">53.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">73.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">52</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">SWE-bench Pro</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">47.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">42.9</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">31.3</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">49.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">35.7</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">SWE-bench Multilingual</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">54.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">52</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">39.7</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">67.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">51.7</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">NL2Repo</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">32.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">27.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">16.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">29.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">15.5</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">SWE Atlas - QnA</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">20.6</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">17.9</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">9.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">15.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td></tr>
<tr><td colspan="6" style="padding:8px 12px;font-weight:600;color:#FD8E5B;border-bottom:1px solid rgba(253, 142, 91, 0.2);background:rgba(253, 142, 91, 0.1)">Reasoning</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">HLE <sub><small>(no tools)</small></sub></td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">20.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">16.8</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">14.7</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">21.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">19.5</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">HLE <sub><small>(with tools)</small></sub></td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">30.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">26.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">24.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">28.9</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">26.5</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">GPQA Diamond</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">86.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">82.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">81.7</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">86</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">84.3</td></tr>
<tr><td colspan="6" style="padding:8px 12px;font-weight:600;color:#FD8E5B;border-bottom:1px solid rgba(253, 142, 91, 0.2);background:rgba(253, 142, 91, 0.1)">Agentic</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">MCP-Atlas</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">54.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">49.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">46.8</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">62.8</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">55</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">Toolathlon-Verified</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">41.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">33.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">29.6</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">41.7</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">52.8</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">WideSearch</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">59.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">55.8</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">53.6</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">60.1</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">54.2</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">BrowseComp</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">56.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">44.8</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">41.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">62</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">ClawEval</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">66.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">63.1</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">53.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">68.7</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">48.5</td></tr>
</tbody>
</table>
</div>

<div style="font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',Roboto,sans-serif;width:100%;margin:0 auto">
<p style="margin-top:4px;font-size:10px;opacity:0.7">
* All results reported for Ornith-1.5 are averaged over five independent runs.<br/>
* Terminal-Bench 2.1 (Terminus-2): evaluated with the Harbor/Terminus-2 framework, parser=json, temperature=1.0, top_p=1.0, 128K context window. Each run uses a 4-hour timeout with 32 CPU cores and 48GB RAM, averaged over 5 runs. We adjust the Qwen chat template to keep training and inference consistent and modify Harbor to align with vLLM's reasoning_content key.<br/>
* Terminal-Bench 2.1 (Claude Code): evaluated with Claude Code 2.1.126, parser=json, temperature=1.0, top_p=1.0, max_new_tokens=131072, averaged over 5 runs (Qwen chat template likewise modified).<br/>
* SWE-bench Verified / Pro / Multilingual: OpenHands harness, temp=1.0, top_p=0.95, 256K context window. Anti-hacking safeguards applied throughout: Git history is removed from the local repository image to prevent access to prior solutions or commits, and network access is disabled.<br/>
* DeepSWE: Claude Code harness, temperature=1.0, top_p=0.95, 256K context window.<br/>
* SWE Atlas QnA / RF / TW: mini-SWE-agent harness, temp=1.0, top_p=0.95, 128K context window, averaged over 5 runs.<br/>
* NL2Repo: temperature=1.0, top_p=1.0, 400K context, 48K output. Access to specified GitHub repositories and pip packages is blocked to prevent reward hacking.<br/>
* HLE: evaluated using Claude 4.6 Opus as the judge model.<br/>
* MCP-Atlas: all models evaluated in thinking mode on the 500-task public subset, 10-minute timeout per task, Claude 4.8 Opus as the judge model.<br/>
* Toolathlon-Verified: official evaluation service with the maximum token limit set to 128K.<br/>
* ClawEval: an agentic code benchmark over real-user task distributions; temp=0.6, 256K context.
</p>
</div>

---

# Ornith-1.0

Ornith-1.0 is the first generation of self-improving open-source models for agentic coding.

Highlights:

- **State-of-the-Art Coding Agents**: Available in 9B-Dense, 31B-Dense, 35B-MoE, and 397B-MoE (post-trained on top of Gemma 4 and Qwen 3.5), achieving state-of-the-art performance among open-source models of comparable size on coding benchmarks such as Terminal-Bench 2.1, SWE-Bench, NL2Repo and OpenClaw.
- **Self-Improving Training Framework**: Ornith-1.0 employs RL to learn to generate not only solution rollouts, but also the scaffold that drives those rollouts. By jointly optimizing the scaffold and the resulting solution, the model discovers better search trajectories and generates higher-quality solutions.
- **Licence**: MIT licensed, globally accessible, and free from regional limitations.

<img style="width: 100%; max-width: 900px;" src="assets/ornith_397b_eval.png" alt="Ornith 397B Benchmark Results" title="Ornith 397B Benchmark Results">


## Ornith-1.0 Benchmarks

Each model is evaluated against its size-appropriate baselines. All three use the same harnesses and decoding setup (see the notes under the tables).

### Ornith-1.0-9B

<div style="font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',Roboto,sans-serif;width:100%;margin:0 auto;padding:16px 0">
<table style="width:100%;table-layout:fixed;border-collapse:collapse;font-size:13px">
<thead><tr>
<th style="width:28%;padding:10px 7px;text-align:left;font-weight:600;border-bottom:2px solid #FD8E5B;color:#FD8E5B"></th>
<th style="width:14.40%;padding:10px 7px;text-align:center;font-weight:700;border-bottom:2px solid #FD8E5B;color:#FD8E5B;font-size:14px;background:rgba(253, 142, 91, 0.12)">Ornith-1.0-9B</th>
<th style="width:14.40%;padding:10px 7px;text-align:center;font-weight:500;border-bottom:2px solid #FD8E5B;color:#FD8E5B;font-size:14px;">Qwen3.5-9B</th>
<th style="width:14.40%;padding:10px 7px;text-align:center;font-weight:500;border-bottom:2px solid #FD8E5B;color:#FD8E5B;font-size:14px;">Qwen3.5-35B</th>
<th style="width:14.40%;padding:10px 7px;text-align:center;font-weight:500;border-bottom:2px solid #FD8E5B;color:#FD8E5B;font-size:14px;">Gemma4-12B</th>
<th style="width:14.40%;padding:10px 7px;text-align:center;font-weight:500;border-bottom:2px solid #FD8E5B;color:#FD8E5B;font-size:14px;">Gemma4-31B</th>
</tr></thead>
<tbody>
<tr><td colspan="6" style="padding:8px 12px;font-weight:600;color:#FD8E5B;border-bottom:1px solid rgba(253, 142, 91, 0.2);background:rgba(253, 142, 91, 0.1)">Agentic Coding</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">Terminal-Bench 2.1 <sub><small>(Terminus-2)</small></sub></td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">43.1</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">21.3</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">41.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">21</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">42.1</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">Terminal-Bench 2.1 <sub><small>(Claude Code)</small></sub></td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">40.6</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">18.9</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">38.9</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">SWE-bench Verified</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">69.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">53.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">70</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">44.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">52</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">SWE-bench Pro</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">42.9</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">31.3</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">44.6</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">27.6</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">35.7</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">SWE-bench Multilingual</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">52</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">39.7</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">60.3</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">32.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">51.7</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">NL2Repo</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">27.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">16.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">20.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">10.3</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">15.5</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">Claw-eval Avg</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">63.1</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">53.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">65.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">32.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">48.5</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">SWE Atlas - QnA</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">17.9</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">9.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">13.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">SWE Atlas - RF</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">16.6</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">4.3</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">10.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">SWE Atlas - TW</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">15.3</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">4.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">9.8</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td></tr>
</tbody>
</table>
</div>

### Ornith-1.0-35B

<div style="font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',Roboto,sans-serif;width:100%;margin:0 auto;padding:16px 0">
<table style="width:100%;table-layout:fixed;border-collapse:collapse;font-size:13px">
<thead><tr>
<th style="width:28%;padding:10px 7px;text-align:left;font-weight:600;border-bottom:2px solid #FD8E5B;color:#FD8E5B"></th>
<th style="width:14.40%;padding:10px 7px;text-align:center;font-weight:700;border-bottom:2px solid #FD8E5B;color:#FD8E5B;font-size:14px;background:rgba(253, 142, 91, 0.12)">Ornith-1.0-35B</th>
<th style="width:14.40%;padding:10px 7px;text-align:center;font-weight:500;border-bottom:2px solid #FD8E5B;color:#FD8E5B;font-size:14px;">Qwen3.5-35B</th>
<th style="width:14.40%;padding:10px 7px;text-align:center;font-weight:500;border-bottom:2px solid #FD8E5B;color:#FD8E5B;font-size:14px;">Qwen3.6-35B</th>
<th style="width:14.40%;padding:10px 7px;text-align:center;font-weight:500;border-bottom:2px solid #FD8E5B;color:#FD8E5B;font-size:14px;">Gemma4-31B</th>
<th style="width:14.40%;padding:10px 7px;text-align:center;font-weight:500;border-bottom:2px solid #FD8E5B;color:#FD8E5B;font-size:14px;">Qwen3.5-397B</th>
</tr></thead>
<tbody>
<tr><td colspan="6" style="padding:8px 12px;font-weight:600;color:#FD8E5B;border-bottom:1px solid rgba(253, 142, 91, 0.2);background:rgba(253, 142, 91, 0.1)">Agentic Coding</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">Terminal-Bench 2.1 <sub><small>(Terminus-2)</small></sub></td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">64.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">41.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">52.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">42.1</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">53.5</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">Terminal-Bench 2.1 <sub><small>(Claude Code)</small></sub></td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">62.8</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">38.9</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">49.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">48.6</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">SWE-bench Verified</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">75.6</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">70</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">73.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">52</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">76.4</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">SWE-bench Pro</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">50.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">44.6</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">49.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">35.7</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">51.6</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">SWE-bench Multilingual</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">69.3</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">60.3</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">67.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">51.7</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">69.3</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">NL2Repo</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">34.6</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">20.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">29.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">15.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">36.8</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">Claw-eval Avg</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">69.8</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">65.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">68.7</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">48.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">70.7</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">SWE Atlas - QnA</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">37.1</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">13.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">15.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">20.4</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">SWE Atlas - RF</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">29.7</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">10.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">11.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">18.4</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">SWE Atlas - TW</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">27.8</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">9.8</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">13.3</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">18.5</td></tr>
</tbody>
</table>
</div>

### Ornith-1.0-397B

<div style="font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',Roboto,sans-serif;width:100%;margin:0 auto;padding:16px 0">
<table style="width:100%;table-layout:fixed;border-collapse:collapse;font-size:13px">
<thead><tr>
<th style="width:24%;padding:10px 7px;text-align:left;font-weight:600;border-bottom:2px solid #FD8E5B;color:#FD8E5B"></th>
<th style="width:9.50%;padding:10px 7px;text-align:center;font-weight:700;border-bottom:2px solid #FD8E5B;color:#FD8E5B;font-size:14px;background:rgba(253, 142, 91, 0.12)">Ornith-1.0-397B</th>
<th style="width:9.50%;padding:10px 7px;text-align:center;font-weight:500;border-bottom:2px solid #FD8E5B;color:#FD8E5B;font-size:14px;">Qwen3.5-397B</th>
<th style="width:9.50%;padding:10px 7px;text-align:center;font-weight:500;border-bottom:2px solid #FD8E5B;color:#FD8E5B;font-size:14px;">Qwen3.7-Max</th>
<th style="width:9.50%;padding:10px 7px;text-align:center;font-weight:500;border-bottom:2px solid #FD8E5B;color:#FD8E5B;font-size:14px;">GLM-5.2-744B</th>
<th style="width:9.50%;padding:10px 7px;text-align:center;font-weight:500;border-bottom:2px solid #FD8E5B;color:#FD8E5B;font-size:14px;">Minimax-M3-428B</th>
<th style="width:9.50%;padding:10px 7px;text-align:center;font-weight:500;border-bottom:2px solid #FD8E5B;color:#FD8E5B;font-size:14px;">DeepSeek-V4-Pro-1.6T</th>
<th style="width:9.50%;padding:10px 7px;text-align:center;font-weight:500;border-bottom:2px solid #FD8E5B;color:#FD8E5B;font-size:14px;">Claude Opus 4.7</th>
<th style="width:9.50%;padding:10px 7px;text-align:center;font-weight:500;border-bottom:2px solid #FD8E5B;color:#FD8E5B;font-size:14px;">Claude Opus 4.8</th>
</tr></thead>
<tbody>
<tr><td colspan="9" style="padding:8px 12px;font-weight:600;color:#FD8E5B;border-bottom:1px solid rgba(253, 142, 91, 0.2);background:rgba(253, 142, 91, 0.1)">Agentic Coding</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">Terminal-Bench 2.1 <sub><small>(Terminus-2)</small></sub></td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">77.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">53.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">73.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">81.0</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">64</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">64</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">70.3</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">85</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">Terminal-Bench 2.1 <sub><small>(Claude Code)</small></sub></td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">78.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">48.6</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">69.8</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">82.7</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">66.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">69.7</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">78.9</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">SWE-bench Verified</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">82.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">76.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">80.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">80.6</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">80.8</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">87.6</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">SWE-bench Pro</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">62.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">51.6</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">60.6</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">62.1</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">59</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">55.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">64.3</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">69.2</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">SWE-bench Multilingual</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">78.9</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">69.3</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">78.3</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">76.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">NL2Repo</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">48.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">36.8</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">47.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">48.9</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">42.1</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">69.7</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">Claw-eval Avg</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">77.1</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">70.7</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">65.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">75.8</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">78.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">SWE Atlas - QnA</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">41.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">20.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">37.9</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">27.2</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">40.3</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">48.8</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">SWE Atlas - RF</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">42.6</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">18.4</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">48.6</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">46.7</td></tr>
<tr><td style="padding:7px 7px;padding-left:20px;border-bottom:1px solid rgba(128, 128, 128, 0.15);">SWE Atlas - TW</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15);font-weight:600;color:#FD8E5B;background:rgba(253, 142, 91, 0.06)">39.1</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">18.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">30.8</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">38.5</td><td style="padding:7px 7px;text-align:center;border-bottom:1px solid rgba(128, 128, 128, 0.15)">-</td></tr>
</tbody>
</table>
</div>

<div style="font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',Roboto,sans-serif;width:100%;margin:0 auto">
<p style="margin-top:4px;font-size:10px;opacity:0.7">
* Terminal-Bench 2.1 (Terminus-2): evaluated with the Harbor/Terminus-2 framework, parser=json, temperature=1.0, top_p=1.0, 128K context window. Each run uses a 4-hour timeout with 32 CPU cores and 48GB RAM, averaged over 5 runs. We adjust the Qwen chat template to keep training and inference consistent and modify Harbor to align with vLLM's reasoning_content key.<br/>
* Terminal-Bench 2.1 (Claude Code): evaluated with Claude Code 2.1.126, parser=json, temperature=1.0, top_p=1.0, max_new_tokens=131072, averaged over 5 runs (Qwen chat template likewise modified).<br/>
* SWE-bench Verified / Pro / Multilingual: OpenHands harness, temp=1.0, top_p=0.95, 256K context window.<br/>
* SWE Atlas QnA / RF / TW: mini-SWE-agent harness, temp=1.0, top_p=0.95, 128K context window, averaged over 5 runs.<br/>
* NL2Repo: temperature=1.0, top_p=1.0, 400K context, 48K output, anti-hacking filters.<br/>
* ClawEval: an agentic code benchmark over real-user task distributions; temp=0.6, 256K context.
</p>
</div>

---

## Quickstart

> **NOTE**
>
> **Ornith models (1.0 and 1.5) are reasoning models**: by default the assistant turn opens with a `<think> … </think>` block before the final answer. The serving recipes below enable a reasoning parser so the chain-of-thought is returned in a separate `reasoning_content` field, and a tool-call parser so the model's `<tool_call>` blocks are surfaced as OpenAI-style `tool_calls`.
>
> Serving Ornith requires recent runtimes:
>
> - **Transformers** ≥ 5.8.1
> - **vLLM** ≥ 0.19.1
> - **SGLang** ≥ 0.5.9
>
> Recommended sampling parameters: `temperature=0.6`, `top_p=0.95`, `top_k=20` (use `temperature=1.0` to reproduce the reported benchmark setup).


### Serving Ornith

Both generations ship as a dense **9B** model plus two **Mixture-of-Experts** models (**35B**, **397B**); Ornith-1.5 additionally provides a quantized **9B-Mobile** build for on-device deployment. All checkpoints expose the same OpenAI-compatible interface and support a **256K (262,144-token) context window**; the dense 9B fits on a single 80GB GPU, while the MoE checkpoints are sharded across a multi-GPU node with tensor parallelism.

**Ornith-1.5 checkpoints (latest):**

| Checkpoint | Architecture | Format | Best for |
|---|---|---|---|
| [Ornith-1.5-9B](https://huggingface.co/deepreinforce-ai/Ornith-1.5-9B) | Dense (~9B) | bf16 | Single-GPU serving & fine-tuning |
| [Ornith-1.5-9B-Mobile](https://huggingface.co/deepreinforce-ai/Ornith-1.5-9B-Mobile) | Dense (~9B) | Quantized | On-device inference on iPhone / Android |
| [Ornith-1.5-35B](https://huggingface.co/deepreinforce-ai/Ornith-1.5-35B) | MoE (35B-A3B) | bf16 | Full-precision multi-GPU serving |
| [Ornith-1.5-397B](https://huggingface.co/deepreinforce-ai/Ornith-1.5-397B) | MoE (397B) | bf16 | Full-precision serving on a multi-GPU node |

**Ornith-1.0 checkpoints:**

| Checkpoint | Architecture | Format | Best for |
|---|---|---|---|
| [Ornith-1.0-9B](https://huggingface.co/deepreinforce-ai/Ornith-1.0-9B) | Dense (~9B) | bf16 | Single-GPU serving & fine-tuning |
| [Ornith-1.0-9B-GGUF](https://huggingface.co/deepreinforce-ai/Ornith-1.0-9B-GGUF) | Dense (~9B) | GGUF (quantized) | Local inference via llama.cpp / Ollama |
| [Ornith-1.0-35B](https://huggingface.co/deepreinforce-ai/Ornith-1.0-35B) | MoE (35B) | bf16 | Full-precision multi-GPU serving |
| [Ornith-1.0-35B-FP8](https://huggingface.co/deepreinforce-ai/Ornith-1.0-35B-FP8) | MoE (35B) | FP8 | ~Half the VRAM on FP8-capable GPUs |
| [Ornith-1.0-35B-GGUF](https://huggingface.co/deepreinforce-ai/Ornith-1.0-35B-GGUF) | MoE (35B) | GGUF (quantized) | Local inference via llama.cpp / Ollama |
| [Ornith-1.0-397B](https://huggingface.co/deepreinforce-ai/Ornith-1.0-397B) | MoE (397B) | bf16 | Full-precision serving on a multi-GPU node |
| [Ornith-1.0-397B-FP8](https://huggingface.co/deepreinforce-ai/Ornith-1.0-397B-FP8) | MoE (397B) | FP8 | Memory-efficient serving on FP8-capable GPUs |

The recipes below stand up an OpenAI-compatible server under the shared alias `Ornith-1.5` (swap `1.5` for `1.0` everywhere to serve the previous generation). Set `MODEL` to the checkpoint you want, and match `--tensor-parallel-size` / `--tp` to your GPU count.

#### vLLM

```bash
# Pick a checkpoint — dense 9B, or MoE 35B / 397B (use Ornith-1.0-*-FP8 for lower-VRAM serving):
MODEL=deepreinforce-ai/Ornith-1.5-397B

# MoE checkpoints (35B / 397B): shard across the node with tensor parallelism.
# Dense checkpoint (9B): fits on a single 80GB GPU — drop --tensor-parallel-size.
vllm serve $MODEL \
    --served-model-name Ornith-1.5 \
    --tensor-parallel-size 8 \
    --host 0.0.0.0 --port 8000 \
    --max-model-len 262144 \
    --gpu-memory-utilization 0.90 \
    --enable-prefix-caching \
    --enable-auto-tool-choice --tool-call-parser qwen3_xml \
    --reasoning-parser qwen3 \
    --trust-remote-code
```

#### SGLang

```bash
# Pick a checkpoint — dense 9B, or MoE 35B / 397B (use Ornith-1.0-*-FP8 for lower-VRAM serving):
MODEL=deepreinforce-ai/Ornith-1.5-397B

# MoE checkpoints (35B / 397B): shard with --tp ; dense 9B: drop --tp for a single GPU.
python -m sglang.launch_server \
    --model-path $MODEL \
    --served-model-name Ornith-1.5 \
    --tp 8 \
    --host 0.0.0.0 --port 8000 \
    --context-length 262144 \
    --mem-fraction-static 0.85 \
    --tool-call-parser qwen3_coder \
    --reasoning-parser qwen3
```

#### Hugging Face Transformers

For a quick local test (or to script offline generation), load the model directly with Transformers. Make sure you have a recent release installed — see the [Transformers installation guide](https://huggingface.co/docs/transformers/installation); Ornith requires `transformers >= 5.8.1`. The dense 9B checkpoint is the easiest to run locally.

```python
from transformers import AutoModelForCausalLM, AutoTokenizer

model_name = "deepreinforce-ai/Ornith-1.5-9B"  # or -35B / -397B, or Ornith-1.0-*

tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForCausalLM.from_pretrained(
    model_name,
    dtype="auto",
    device_map="auto",
)

messages = [
    {"role": "user", "content": "Write a Python function is_prime(n). Keep it short."}
]
text = tokenizer.apply_chat_template(
    messages,
    tokenize=False,
    add_generation_prompt=True,
)

inputs = tokenizer(text, return_tensors="pt").to(model.device)
generated = model.generate(
    **inputs,
    max_new_tokens=512,
    do_sample=True,
    temperature=0.6,
    top_p=0.95,
    top_k=20,
)
output_ids = generated[0][inputs.input_ids.shape[1]:]

# The reply contains a <think> ... </think> reasoning block followed by the answer.
content = tokenizer.decode(output_ids, skip_special_tokens=True)
print(content)
```

To split the reasoning trace from the final answer, parse on the `</think>` marker:

```python
text = tokenizer.decode(output_ids, skip_special_tokens=True)
if "</think>" in text:
    reasoning, answer = text.split("</think>", 1)
    reasoning = reasoning.replace("<think>", "").strip()
    answer = answer.strip()
else:
    reasoning, answer = "", text.strip()
```

### Using Ornith via the Chat Completions API

Once a vLLM or SGLang server is running, talk to it with any OpenAI-compatible client.

#### Basic Usage

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="EMPTY",  # any non-empty string works for a local server
)

response = client.chat.completions.create(
    model="Ornith-1.5",
    messages=[
        {"role": "user", "content": "Write a one-line Python lambda that squares a number."}
    ],
    temperature=0.6,
    top_p=0.95,
    max_tokens=1024,
)

message = response.choices[0].message
# reasoning_content holds the <think> trace; content holds the final answer.
print("reasoning:", getattr(message, "reasoning_content", None))
print("answer:", message.content)
```

You can also stream tokens, or hand the model tools — Ornith emits well-formed function calls that the server parses into the standard `tool_calls` field:

```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "Get the current weather for a city",
            "parameters": {
                "type": "object",
                "properties": {"city": {"type": "string"}},
                "required": ["city"],
            },
        },
    }
]

response = client.chat.completions.create(
    model="Ornith-1.5",
    messages=[{"role": "user", "content": "What is the weather in Paris right now?"}],
    tools=tools,
    tool_choice="auto",
    temperature=0.6,
    max_tokens=2048,
)

tool_call = response.choices[0].message.tool_calls[0]
print(tool_call.function.name, tool_call.function.arguments)
# -> get_weather {"city": "Paris"}
```

You can point any OpenAI-compatible SDK (Python, Node.js, etc.) or `curl` at the same `/v1/chat/completions` endpoint.

## Agentic Usage

Ornith models excel in tool-calling and agentic coding capabilities.

### Agent Frameworks

Because Ornith exposes an OpenAI-compatible endpoint with tool calling, it works out of the box with standard agent frameworks. Below is a minimal example that connects Ornith to tools through an MCP server.

```python
import os
from openai import OpenAI

client = OpenAI(
    base_url=os.getenv("OPENAI_BASE_URL", "http://localhost:8000/v1"),
    api_key=os.getenv("OPENAI_API_KEY", "EMPTY"),
)

tools = [
    {
        "type": "function",
        "function": {
            "name": "run_shell",
            "description": "Run a shell command and return its output.",
            "parameters": {
                "type": "object",
                "properties": {
                    "command": {"type": "string", "description": "The command to run"}
                },
                "required": ["command"],
            },
        },
    }
]

messages = [{"role": "user", "content": "List the Python files in the current directory."}]

response = client.chat.completions.create(
    model="Ornith-1.5",
    messages=messages,
    tools=tools,
    temperature=0.6,
    top_p=0.95,
)
print(response.choices[0].message)
```

**Examples of using Ornith with agent harness:**

#### Hermes Agent
```bash
# Hermes talks to any OpenAI-compatible endpoint — point it at your Ornith server.
export OPENAI_BASE_URL="http://localhost:8000/v1"
export OPENAI_API_KEY="EMPTY"
export MODEL="Ornith-1.5"
```

#### OpenHands
```bash
pip install openhands-ai

# OpenHands routes through LiteLLM; the "openai/" prefix selects the OpenAI-compatible path.
export LLM_MODEL="openai/Ornith-1.5"
export LLM_BASE_URL="http://localhost:8000/v1"
export LLM_API_KEY="EMPTY"

# Launch the CLI (or run the official OpenHands Docker image with the same env vars).
openhands
```

#### llama.cpp / Ollama
```bash
# Both runtimes load a GGUF build — currently published for the Ornith-1.0 9B and 35B checkpoints (swap -9B for -35B).

# llama.cpp — serve an OpenAI-compatible API on port 8000.
llama-server -hf deepreinforce-ai/Ornith-1.0-9B-GGUF --port 8000 -c 262144

# Ollama — pull and chat with the same GGUF straight from Hugging Face.
ollama run hf.co/deepreinforce-ai/Ornith-1.0-9B-GGUF
```

#### Unsloth Studio

```bash
pip install unsloth

# Load Ornith for fast local inference or fine-tuning (Python):
#   from unsloth import FastLanguageModel
#   model, tokenizer = FastLanguageModel.from_pretrained(
#       "deepreinforce-ai/Ornith-1.5-9B",  # or Ornith-1.0-9B
#       max_seq_length=262144,
#       load_in_4bit=True,
#   )
```


#### OpenClaw

```bash
# OpenClaw talks to any OpenAI-compatible endpoint — point it at your Ornith server.
export OPENAI_BASE_URL="http://localhost:8000/v1"
export OPENAI_API_KEY="EMPTY"
export OPENAI_MODEL="Ornith-1.5"
```


### Coding CLIs

Ornith is optimized for terminal-based coding agents. Point any OpenAI-compatible coding CLI at your Ornith endpoint (set `OPENAI_BASE_URL` and `OPENAI_API_KEY`) to understand large codebases, automate tedious work, and ship faster.

#### OpenCode
```bash
# Register your local Ornith endpoint as a provider in ~/.config/opencode/opencode.json:
#
# {
#   "$schema": "https://opencode.ai/config.json",
#   "provider": {
#     "ornith": {
#       "npm": "@ai-sdk/openai-compatible",
#       "name": "Ornith (local)",
#       "options": { "baseURL": "http://localhost:8000/v1", "apiKey": "EMPTY" },
#       "models": { "Ornith-1.5": { "name": "Ornith-1.5" } }
#     }
#   }
# }

opencode
```

### Citation

If you find our work helpful, feel free to give us a cite.

```bibtex
@misc{ornith-1.5,
    title = {{Ornith-1.5}: From Self-Scaffolding to Self-Improvement},
    url = {https://ornith.ai/ornith_1_5.html},
    author = {{Ornith Team}},
    year = {2026}
}

@misc{ornith-1.0,
    title = {{Ornith-1.0}: Agentic Coding, Open to All},
    url = {https://ornith.ai/ornith_1_0.html},
    author = {{Ornith Team}},
    year = {2026}
}
```
