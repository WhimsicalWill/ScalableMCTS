---
layout: distill
title: Understanding Methods for Scalable MCTS
description: Monte Carlo Tree Search (MCTS) is pivotal for decision-making in environments with large state spaces, such as strategic games and real-world optimization. This blogpost explores scalable methods for parallelizing MCTS through various forms of parallelism and distributed approaches. By analyzing techniques such as leaf, root, and tree parallelism, as well as advanced strategies like lock-free concurrency, virtual loss, and distributed depth-first methods, we explore the challenges and trade-offs of parallelizing MCTS effectively.
date: 2025-04-28
future: true
htmlwidgets: true
hidden: false

# Anonymize when submitting
# authors:
#   - name: Anonymous

authors:
    - name: Anonymous

# must be the exact same name as your blogpost
bibliography: 2025-04-28-scalable-mcts.bib

# Add a table of contents to your post.
#   - make sure that TOC names match the actual section names
#     for hyperlinks within the post to work correctly.
#   - please use this format rather than manually creating a markdown table of contents.
toc:
    - name: Intro
    - name: MCTS Background
      subsections:
          - name: The Four Phases of MCTS
          - name: The UCT Selection Policy
    - name: How can we scale MCTS?
      subsections:
          - name: Leaf Parallelism
          - name: Root Parallelism
          - name: Tree Parallelism
    - name: Lock-Based and Lock-Free Tree Parallelism
    - name: Virtual Loss
    - name: Transposition Driven Scheduling
    - name: Distributed MCTS
    - name: Distributed Depth-First MCTS
    - name: Conclusion

# Below is an example of injecting additional post-specific styles.
# This is used in the 'Layouts' section of this post.
# If you use this post as a template, delete this _styles block.
_styles: >
    .fake-img {
      background: #bbb;
      border: 1px solid rgba(0, 0, 0, 0.1);
      box-shadow: 0 0px 4px rgba(0, 0, 0, 0.1);
      margin-bottom: 12px;
    }
    .fake-img p {
      font-family: monospace;
      color: white;
      text-align: left;
      margin: 12px 0;
      text-align: center;
      font-size: 16px;
    }
---

## Intro

Recently, there has been a large focus on training Large Language Models (LLMs) with up to trillions of parameters, using vast amounts of compute for training. However, expecting these models to produce perfect answers instantaneously -- especially for complex queries -- is unrealistic. The AI industry is now shifting its focus to optimizing inference-time compute, seeking ways to harness computational resources more effectively. One promising approach is leveraging scalable search algorithms, which enable models to plan, reason, and refine their outputs. With enough compute, these methods have the potential to solve many of humanity's grand challenges.

Rich Sutton’s [_bitter lesson_](http://www.incompleteideas.net/IncIdeas/BitterLesson.html) highlights why scalability is key:

> “One thing that should be learned from the bitter lesson is the great power of general-purpose methods, of methods that continue to scale with increased computation even as the available computation becomes very great. The two methods that seem to scale arbitrarily in this way are **search** and **learning**.”

One method that exemplifies this principle is Monte Carlo Tree Search (MCTS), demonstrating remarkable results with increased computation. This is evident in the performance of DeepMind's MuZero in the game of Go, where MCTS-based planning achieves significant improvements as the number of simulations grows:

<div class="col-sm mt-3 mt-md-0">
    {% include figure.html path="assets/img/2025-04-28-scalable-mcts/mu_zero_scaling.png" class="img-fluid rounded z-depth-1" %}
</div>
<div class="caption">
Additional search time improves MuZero's performance in Go (for both real and learned simulators). Adapted from Schrittwieser et al.<d-cite key="schrittwieser2020mastering"></d-cite>
</div>

MuZero's success reveals a characteristic relationship between search time and performance for the MCTS algorithm: performance typically improves logarithmically, where each doubling of computational resources adds a relatively constant increment to playing strength<d-cite key="camacho2017mcts"></d-cite>. The scalability of MuZero is also driven by techniques like tree parallelism and virtual loss, which we will cover in more detail after introducing the fundamentals of MCTS.

## MCTS Background

In this blog post, we will focus on MCTS with the Upper Confidence bounds applied to Trees (UCT) algorithm, which is a widely used variant that effectively balances exploration and exploitation. For simplicity, we will refer to this variant as "MCTS" throughout the post. MCTS, a powerful algorithm for decision making in large state spaces, is commonly used in games, optimization problems, and real-world domains such as protein folding and molecular design. It stands out for its ability to search complex spaces without the need for additional heuristic knowledge, making it adaptable across a variety of problems. Before MCTS became prominent, techniques like minimax with alpha-beta pruning were the standard in game AI. While alpha-beta pruning could efficiently reduce the search space, its effectiveness often depended on the quality of evaluation functions and move ordering. MCTS offered a different approach that could work without domain-specific knowledge, though both methods can benefit from incorporating heuristics<d-cite key="swiechowski2022mcts"></d-cite>. Below, we will walk through the details of MCTS and UCT.

### The Four Phases of MCTS

MCTS operates in an environment that defines the states, actions, transition rules, and reward assignments. The algorithm iteratively builds a search tree, collecting statistics for each node, such as visit counts and the average reward from simulations that pass through it. These statistics guide the decision-making process, enabling the algorithm to balance exploration and exploitation. The algorithm operates in four main phases:

<div class="row mt-4">
  <div class="col-md-6">
    <h4>1. Selection</h4>
    <p>
      Starting from the root, MCTS traverses child nodes using a policy that balances exploration and exploitation until it reaches a node with unexplored children (allowing for expansion) or a terminal node. 
    </p>
  </div>
  <div class="col-md-6">
    {% include figure.html path="assets/img/2025-04-28-scalable-mcts/mcts_phases_selection.svg" class="img-fluid rounded" %}
  </div>
</div>

<div class="row mt-4">
  <div class="col-md-6">
    <h4>2. Expansion</h4>
    <p>
      If the selected node is not terminal, the algorithm chooses an unexplored child node randomly and adds it to the tree.
    </p>
  </div>
  <div class="col-md-6">
    {% include figure.html path="assets/img/2025-04-28-scalable-mcts/mcts_phases_expansion.svg" class="img-fluid rounded" %}
  </div>
</div>

<div class="row mt-4">
  <div class="col-md-6">
    <h4>3. Simulation</h4>
    <p>
      From the newly expanded node, a simulation (or 'rollout') is performed by sampling actions using a default (often random) policy until a terminal state is reached. The cumulative reward obtained during this phase serves as feedback for evaluating the node.
    </p>
  </div>
  <div class="col-md-6">
    {% include figure.html path="assets/img/2025-04-28-scalable-mcts/mcts_phases_simulation.svg" class="img-fluid rounded" %}
  </div>
</div>

<div class="row mt-4">
  <div class="col-md-6">
    <h4>4. Backpropagation</h4>
    <p>
      Once a terminal state is reached, the simulation's result is propagated back up the tree. 
      At each node along the path, visit counts are incremented, and average rewards are updated to reflect the outcome.
    </p>
  </div>
  <div class="col-md-6">
    {% include figure.html path="assets/img/2025-04-28-scalable-mcts/mcts_phases_backpropagation.svg" class="img-fluid rounded" %}
  </div>
</div>

<div class="caption mt-3">
  These diagrams and their descriptions outline the four main phases of MCTS: Selection, Expansion, Simulation, and Backpropagation. Diagrams adapted from Wikipedia<d-cite key="wiki:mcts"></d-cite>.
</div>

MCTS is an **anytime** algorithm, meaning it can be stopped at any point during its execution and still return the best decision found up to that point. This is particularly useful when dealing with problems where the state space is too large to be fully explored. In practical applications, MCTS operates within a computational budget, which could be defined by a fixed number of iterations or a set amount of time.

At the conclusion of the MCTS process, the algorithm recommends the action $a_{final}$ that maximizes the average reward $Q(s_0, a)$ among all possible actions $a$ from the root state $s_0$:

$$
a_{final} = \underset{a \in A(s_0)}{\operatorname{argmax}} Q(s_0, a) \tag{1}
$$

where:

-   $A(s_0)$ represents the set of actions available in the root state $s_0$.
-   $Q(s_0, a)$ represents the average reward from playing action $a$ in state $s_0$ based on simulations performed so far.

As the number of iterations grows to infinity, the average reward estimates $Q(s_0, a)$ for each of the actions from the root state converge in probability to their minimax optimal values<d-cite key="kocsis2006bandit"></d-cite>. Consequently, as MCTS performs more iterations, the selected action $a_{final}$ converges to the optimal action $a^*$.

### The UCT Selection Policy

A key component of MCTS is the _Upper Confidence Bounds for Trees (UCT)_ algorithm, introduced by Kocsis and Szepesvári<d-cite key="kocsis2006bandit"></d-cite>. This algorithm applies the principle of _optimism in the face of uncertainty_ to balance between exploration (of not well-tested actions) and exploitation (of the best actions identified so far) while building the search tree. UCT extends the UCB1 algorithm, adapting it for decision-making within the tree during the selection phase<d-cite key="swiechowski2022mcts"></d-cite>.

During the selection phase, the algorithm chooses actions based on the statistics of actions that have already been explored within the search tree. The action $a_{selection}$ to play in a given state $s$ is selected by maximizing the UCT score shown in brackets below:

$$
a_{selection} = \underset{a \in A(s)}{\operatorname{argmax}} \left\{ Q(s, a) + C \sqrt{\frac{\ln N(s)}{N(s, a)}} \right\} \tag{2}
$$

where:

-   $a_{selection}$ is the action selected from state $s$.
-   $A(s)$ is the set of actions available in state $s$.
-   $Q(s, a)$ represents the average reward from playing action $a$ in state $s$ from simulations performed so far.
-   $N(s)$ is the number of times state $s$ has been visited.
-   $N(s, a)$ is the number of times action $a$ has been played from state $s$.
-   $C$ is a constant controlling the balance between exploration and exploitation. In general, it is set differently depending on the domain.

This concludes our discussion of the foundational elements of MCTS, providing the necessary context for understanding scalable MCTS. For a deeper dive into MCTS, I highly recommend [this guide](https://gibberblot.github.io/rl-notes/single-agent/mcts.html "Monte Carlo Tree Search (MCTS)") by Tim Miller.

## How can we scale MCTS?

Its great that MCTS improves its solution quality with time, but what if we have an urgent problem that must be solved within a short amount of time (e.g. real-time decision making)? In this case, MCTS can be adapted to leverage parallel computation, enabling us to find a better solution within our fixed time budget. As we will see, parallelizing MCTS without degrading its performance is challenging since each iteration requires information from all previous iterations to provide an effective **exploration-exploitation tradeoff**<d-cite key="liu2020watch"></d-cite>. In this blogpost, we will explore scalable adaptations of MCTS that effectively parallelize and distribute its computation.

To find better solutions without increasing [wall-clock time](https://en.wikipedia.org/wiki/Elapsed_real_time), we can consider how the four different phases of MCTS can be distributed. There are three broad types of parallelism used to scale MCTS, and we will discuss each one below.

{% include figure.html path="assets/img/2025-04-28-scalable-mcts/mcts_parallelism.png" class="img-fluid" %}

<div class="caption">
  Diagram comparing various approaches to parallelizing MCTS. From Chaslot et al.<d-cite key="chaslot2008parallel"></d-cite>
</div>

### Leaf Parallelism

Leaf parallelism focuses on the simulation phase of MCTS. By distributing the rollout simulations across multiple workers, it achieves a high degree of parallelism with minimal coordination overhead. Once we expand a new leaf node, we execute rollouts on multiple different workers, getting more rollout results in the same amount of wall-clock time. Leaf parallelism reduces the variance of our value estimates, giving us more precise estimates of the true value of a given action. Leaf parallelism requires no communication between worker machines until the results of the rollouts are aggregated. Although leaf parallelism increases the speed of variance reduction, its effectiveness can be limited by the quality of rollouts. If early simulations indicate a low value, running more simulations can be a complete waste of compute. Chaslot et al. find that the strength of leaf parallelism is rather low and suggest that it is not a good way to parallelize MCTS<d-cite key="chaslot2008parallel"></d-cite>.

### Root Parallelism

Root parallelism takes the approach of distributing all phases onto separate worker machines. Each worker uses MCTS to build a **separate** tree concurrently. This approach is straightforward and avoids the complexities associated with managing shared memory. It is analogous to ensemble methods in machine learning where multiple models (trees, in this case) vote on the best action. Each tree explores different trajectories independently, which can provide a broader exploration of the state space but may result in redundant computations<d-cite key="chaslot2008parallel"></d-cite>. The independence of each tree allows for diverse explorations of the search space, increasing the likelihood of escaping local optima that a single search might get trapped in<d-cite key="chaslot2008parallel"></d-cite>. When the available time is spent, we aggregate the results of the different trees and select the most promising action based on a majority vote or averaging method. While root parallelism is easy to implement and requires little communication, it often lacks efficiency because workers build completely separate trees with no information sharing during the search process, preventing them from leveraging information gained by other workers.

### Tree Parallelism

Tree parallelism, which will be the focus for the remainder of this blogpost, is widely used in modern distributed MCTS implementations and is the method of choice in the AlphaGo line of work<d-cite key="schrittwieser2024questions"></d-cite>. It shares information effectively by building out a single tree using multiple threads. Further, it accelerates tree expansion by allowing concurrent expansions on different parts of the tree. The primary implementation challenge lies in striking a balance between maximizing computational resource utilization and ensuring sufficient focus on the most promising parts of the tree.

**There are many compelling reasons to use tree parallelism:**

-   It may find a better solution within a fixed amount of wall-clock time.
-   It explores of more of the game tree, increasing the likelihood of discovering the optimal trajectory (sequence of actions).
-   It makes full use of accelerators (GPUs, TPUs, etc.).

**Yet at the same time, there are many challenges:**

-   MCTS relies on the statistics gathered from previous iterations to guide future decisions. In parallel implementations, these statistics may become outdated due to concurrent updates, potentially leading to suboptimal traversal of the search tree.
-   Evaluating actions that are far away from the optimal trajectory can be useless or even harmful, as approximation errors and variability in rollouts can make bad moves seem promising in the short term<d-cite key="schrittwieser2024questions"></d-cite>.
-   Managing concurrent access to the tree and ensuring data integrity introduces complexity. For instance, race conditions or data overwrites could corrupt the search process<d-cite key="steinmetz2020more"></d-cite>.

To address these challenges, techniques such as locking, virtual loss, and transposition driven scheduling have been proposed. Below, we will expand on these different techniques and explain how they are used to implement tree parallelism effectively.

## Lock-Based and Lock-Free Tree Parallelism

As mentioned, one of the main challenges with tree parallelism is dealing with resource contention. There are two main lock-based approaches used in the literature to address this challenge: global mutex and local mutex methods.

Global mutex methods use a single lock to control access to the tree during critical operations. While threads must acquire this lock during selection, expansion, and backpropagation phases, they can perform simulations without acquiring the lock since they don't modify the tree. This approach is straightforward to implement but creates a significant bottleneck since threads often have to wait before accessing the tree. Consequently, global mutex methods show limited scalability, making them suitable primarily for systems that use few threads.

Local mutex methods make locking much more fine-grained, allowing multiple workers modify the shared tree at the same time. In local mutex methods, threads only lock the node they are currently working on. Since this allows multiple workers to operate on different parts of the shared search tree, local mutex methods have much greater scalability compared to global mutex methods. However, while this method increases parallelism, it also increases overhead since we have to manage multiple locks. Chaslot et al. recommend using fast-access mutexes like spinlocks for implementation, which are designed for situations where threads are expected to hold locks for very short durations<d-cite key="chaslot2008parallel"></d-cite>.

Since lock-based methods add additional overhead and require a complex implementation, there are also lock-free methods that rely on atomic operations and memory models to enable concurrent access without using traditional locks<d-cite key="steinmetz2020more"></d-cite>. While lock-free algorithms can be challenging to implement correctly, they can provide better performance than lock-based approaches while still ensuring thread safety.

## Virtual Loss

There is a glaring issue with naively parallelizing MCTS: if selection is performed in parallel by multiple workers, they will all select the same path through the search tree. One technique often used to fix this is virtual loss, which reduces redundant exploration by dynamically adjusting UCT scores. Virtual loss mitigates this issue by temporarily reducing the UCT score of nodes currently being explored, encouraging workers to diversify their trajectories. Specifically, it temporarily assumes that ongoing rollouts at a given node will result in zero reward, reducing the likelihood of other threads selecting the same node until its rollouts are completed and backpropagated.

To illustrate how virtual loss increases trajectory diversity, consider the following diagram from Yang et al., which compares parallelization with and without virtual loss. In this scenario, we assume that there are multiple workers traversing the search tree at the same time, but they perform selection in some specific order. As a result, virtual loss reduces UCT scores for nodes being explored, enabling subsequent workers to take distinct search paths and avoid redundant computation.

{% include figure.html path="assets/img/2025-04-28-scalable-mcts/virtual_loss_traversal.png" class="img-fluid" %}

<div class="caption">
Comparison of (a) naive parallel UCT, where workers redundantly explore the same path, and (b) parallel UCT with virtual loss, where ongoing rollouts dynamically reduce UCT scores, resulting in distinct search paths for each worker. Workers from earliest to latest are shown in green, red, and blue. Adapted from Yang et al.<d-cite key="yang2021practical"></d-cite>
</div>

Virtual loss modifies the usual UCT selection policy to be the following:

$$
a_{selection} = \underset{a \in A(s)}{\operatorname{argmax}} \left\{ \frac{N(s, a)Q(s, a)}{N(s, a) + T(s, a)} + C \sqrt{\frac{\ln (N(s) + T(s))}{N(s, a) + T(s, a)}} \right\} \tag{3}
$$

where:

-   $a_{selection}$ is the action selected from state $s$.
-   $A(s)$ is the set of actions available in state $s$.
-   $Q(s, a)$ represents the average reward from playing action $a$ in state $s$ based on simulations performed so far.
-   $N(s)$ is the number of times state $s$ has been visited.
-   $N(s, a)$ is the number of times action $a$ has been played from state $s$.
-   $T(s)$ is the number of workers currently exploring action $a$ in state $s$.
-   $T(s, a)$ is the number of workers currently exploring action $a$ in state $s$.
-   $C$ is a constant controlling the balance between exploration and exploitation. In general, it is set differently depending on the domain.

In this modified UCT score, there are two mechanisms that punish nodes having ongoing rollouts. $T(s, a)$ penalizes nodes with many active simulations.
Virtual loss encourages exploration by diversifying worker trajectories, but its fixed penalty can lead to inefficiencies. For example, promising nodes may be overly penalized, hindering exploitation<d-cite key="mirsoleimani2015parallel"></d-cite>.

To overcome the challenges of fixed penalties in virtual loss, Liu et al. proposed an alternative approach called _Watch the Unobserved in UCT (WU-UCT)_. WU-UCT modifies the standard virtual loss formula by maintaining the original average reward, rather than reducing it as in virtual loss. However, it applies the same penalty adjustment to the exploration term, encouraging workers to prioritize less-visited actions. Thus, the action $a_{selection}$ to play in a given state $s$ is selected by maximizing the WU-UCT score shown in brackets below:

$$
a_{selection} = \underset{a \in A(s)}{\operatorname{argmax}} \left\{Q(s, a) + C \sqrt{\frac{\ln (N(s) + T(s))}{N(s, a) + T(s, a)}} \right\} \tag{4}
$$

Just as in virtual loss, $T(s, a)$ penalizes nodes with many active simulations. Unlike virtual loss, which assumes zero reward for ongoing workers, WU-UCT assumes that the reward will be $Q(s, a)$, the average reward observed so far. By preserving average rewards and penalizing only the exploration term, WU-UCT aims to improve the efficiency of parallel rollouts while reducing the likelihood of redundant computations.

Liu et al. find that their system using WU-UCT demonstrated a near-linear speedup as the number of workers increased when testing on an RL benchmark suite of 15 Atari games, with only slight performance degradation compared to sequential MCTS<d-cite key="liu2020watch"></d-cite>. However, despite its theoretical advantages, WU-UCT is not universally superior. Empirical results show that vanilla virtual loss outperforms WU-UCT in some domains (e.g. molecular design)<d-cite key="yang2021practical"></d-cite>

## Distributed MCTS

Scaling MCTS to tackle larger state spaces often requires distributing the tree and its associated statistics across multiple machines. Liu et al. propose a framework that decouples the core phases of MCTS into independent tasks, enabling parallel execution across different worker types: master, expansion, and simulation workers.

-   **Master Worker**: Maintains the central MCTS logic and stores the global data structures, including the search tree, node statistics, and visit counts. It coordinates all tasks and tracks active rollouts.
-   **Expansion Workers**: Traverse the known tree using the UCT selection policy, reporting back paths that end at unexplored leaf nodes. The master uses these paths to update the active rollout counters.
-   **Simulation Workers**: Perform rollouts from the newly expanded leaf nodes to terminal states, generating rewards that are sent back to the master for backpropagation.

The following diagram from Liu et al. illustrates the flow of tasks between the master process and worker machines, showing how expansion and simulation tasks are coordinated in their framework:

{% include figure.html path="assets/img/2025-04-28-scalable-mcts/wu_mcts_diagram.png" class="img-fluid" %}

<div class="caption">
  Flowchart depicting the coordination between the master process, expansion workers, and simulation workers in a distributed MCTS framework. From Liu et al.<d-cite key="liu2020watch"></d-cite>
</div>

By decoupling the expansion phase (traversing the known tree to find a leaf) from the simulation phase (randomly exploring beyond the leaf), this framework effectively parallelizes MCTS. The master tracks rollouts in progress, ensuring updates to node statistics (e.g., visit counts) remain consistent.

While this system involves frequent communication between workers and the master, Liu et al. demonstrate that the computational cost of the expansion and simulation phases significantly outweighs communication overhead. As a result, communication does not become a bottleneck, even with many workers.

## Transposition Driven Scheduling

In 2011, Yoshizoe et al.<d-cite key="yoshizoe2011scalable"></d-cite> introduced Transposition Driven Scheduling (TDS) as an efficient framework for distributing tasks in parallelized MCTS. TDS focuses on evenly partitioning the data across worker machines, minimizing communication overhead, and efficiently coordinating computation without moving data unnecessarily. This approach enables scalable parallelism, even for tasks with large search trees.

At its core, TDS is a method for evenly distributing data across multiple worker machines and coordinating tasks without excessive data movement. Each record of data, such as a node in the search tree, is assigned a "home" worker by applying a hash function. This ensures that the data is partitioned across workers, with each record stored exclusively on its designated worker. Crucially, rather than moving data between workers, TDS minimizes network overhead by sending requests to the worker where the data resides. For example, if Worker A needs to access data housed on Worker B, it forwards a request to Worker B, which processes the request locally and returns the result. This design is very efficient, as moving data across a network is generally far slower than encoding requests to process that data and sending the requests where the data resides.

TDS communication is inherently asynchronous, meaning that when a worker receives a request for data located on another worker, it forwards the request without pausing local computation. The responding worker processes the request and sends the result back. This design maximizes throughput, as frequent communication is offset by asynchronicity, allowing workers to continue processing other tasks while waiting for responses.

Yoshizoe et al.<d-cite key="yoshizoe2011scalable"></d-cite> demonstrated how TDS can support MCTS by building a distributed search tree. Each node in the tree is assigned to a worker using a hash-based mapping. When a new node is discovered, its home worker is determined, and memory is allocated for its metadata (e.g., parent, children, values, and visit counts). Workers handle tasks such as expanding the tree or conducting rollouts, with results propagated back to update statistics. Despite occasional use of outdated data, the authors showed that TDS-based MCTS closely approximates sequential MCTS while achieving significant speedups through parallelism.

While TDS minimizes communication overhead through its asynchronous design, this efficiency introduces tradeoffs. For instance, the reliance on asynchronous communication can result in occasional use of outdated node statistics, making the search process less effective. Further, nodes higher up in the tree, especially the root node, receive disproportionately more requests, leading to workload imbalances. These bottlenecks can slow down processing and reduce scalability when many workers are involved. Understanding these tradeoffs is critical for leveraging TDS effectively in distributed MCTS applications.

## Distributed Depth-First MCTS

To address the uneven distribution of workload in parallel MCTS, Yoshizoe et al.<d-cite key="yoshizoe2011scalable"></d-cite> proposed a depth-first version of MCTS (TDS-df-UCT). The core idea is to reduce the frequency of backpropagation steps, limiting contention around heavily visited nodes, particularly the root. TDS-df-UCT operates similarly to traditional MCTS but delays backpropagation until a node is sufficiently explored. This reduces the number of messages sent between workers and alleviates bottlenecks caused by frequent updates to high-traffic nodes.

In TDS-df-UCT, the tree is distributed across workers using Transposition Driven Scheduling (TDS), where each worker stores a subset of the tree based on a hash function. While this approach minimizes communication overhead and allows for asynchronous computation, the reduction in backpropagation frequency introduces the following drawbacks:

-   **Shallow Trees**: By skipping frequent backpropagations, TDS-df-UCT may overlook discoveries from other workers, leading to trees that are wider and shallower than sequential MCTS.
-   **Delayed Information Sharing**: Since history is carried only in the messages exchanged between workers, new findings take longer to propagate through the tree.

Despite these limitations, TDS-df-UCT achieves significant speedups over sequential MCTS by leveraging distributed resources effectively.

Building upon TDS-df-UCT, Yang et al. introdued Massively Parallel MCTS (MP-MCTS), which refines the parallel MCTS process to address its shortcomings. Introduced in the context of molecular design and other large-scale search problems, MP-MCTS incorporates several key innovations<d-cite key="yang2021practical"></d-cite>:

-   **Node-Level History Tables**: Unlike TDS-df-UCT, where history is carried only in messages, MP-MCTS stores detailed statistical histories (e.g., visit counts and rewards) within each node. This accelerates the dissemination of the latest simulation results across workers and allows for more informed decisions during traversal.
-   **Strategic Backpropagation**: MP-MCTS introduces a more dynamic backpropagation strategy, where updates are performed as needed to maintain accurate UCT value estimates. This prevents over-exploration of less promising branches while ensuring timely propagation of critical information.
-   **Focused Exploration**: By enabling workers to leverage the most up-to-date statistics, MP-MCTS directs rollouts toward deeper, more promising parts of the tree, resulting in higher-quality solutions.

These enhancements enable MP-MCTS to approximate the behavior of sequential MCTS while achieving substantial speedups in distributed environments. The figure below demonstrates how MP-MCTS consistently produces deeper trees compared to TDS-df-UCT and even rivals the depth of non-parallel MCTS when given equivalent computational resources:

{% include figure.html path="assets/img/2025-04-28-scalable-mcts/mp_mcts_depth_comparison.png" class="img-fluid" %}

<div class="caption">
  Depth of nodes in the tree achieved by MP-MCTS compared to TDS-df-UCT and traditional MCTS across different configurations. Adapted from Yang et al.<d-cite key="yang2021practical"></d-cite>
</div>

Experiments show that MP-MCTS not only achieves deeper trees but also consistently outperforms TDS-df-UCT in solution quality. In molecular design benchmarks, MP-MCTS running on 256 cores for 10 minutes found solutions comparable to non-parallel MCTS running for 2560 minutes<d-cite key="yang2021practical"></d-cite>.

Distributed depth-first MCTS approaches like TDS-df-UCT and MP-MCTS represent a significant step forward in scaling MCTS for large-scale distributed environments. While TDS-df-UCT introduced foundational ideas for mitigating communication bottlenecks, MP-MCTS refined these concepts to achieve better tree depth, scalability, and solution quality. By cleverly reducing backpropagation overhead and introducing node-level history tables, these methods achieve significant speedups while approximating the behavior of sequential MCTS.

## Conclusion

MCTS is a foundational algorithm for decision-making, underpinning technologies like MuZero and proving effective in fields that depend on intelligent decision-making. While inherently sequential, MCTS can be scaled effectively through distributed methods that approximate its behavior. Techniques like leaf, root, and tree parallelism, as well as advanced strategies such as lock-free concurrency and virtual loss, enable substantial improvements in computational efficiency, particularly when evaluated within fixed time constraints.

No single distributed MCTS method is universally optimal; each involves trade-offs in communication frequency, message size, and resource allocation. Choosing the right approach requires a thorough understanding of the system's network and computational constraints. Ultimately, scalable search methods like MCTS hold the potential to address some of humanity’s most complex problems -- whether in science, medicine, or engineering -- by unlocking new levels of reasoning and planning through parallelized computation. Understanding the trade-offs of scalable MCTS approaches and knowing when to apply different methods may pave the way for breakthroughs in fields where intelligent decision-making is critical.
