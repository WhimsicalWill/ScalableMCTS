---
layout: distill
title: Understanding Methods for Scalable MCTS
description: Your blog post's abstract.
    Please add your abstract or summary here and not in the main body of your text.
    Do not include math/latex or hyperlinks.
date: 2025-04-28
future: true
htmlwidgets: true
hidden: false

# Anonymize when submitting
# authors:
#   - name: Anonymous

authors:
    - name: Will Knipe
      url: ""
      affiliations:
          name: CMU
    - name: Nate Rising
      url: ""
      affiliations:
          name: CMU

# must be the exact same name as your blogpost
bibliography: 2025-04-28-scalable-mcts.bib

# Add a table of contents to your post.
#   - make sure that TOC names match the actual section names
#     for hyperlinks within the post to work correctly.
#   - please use this format rather than manually creating a markdown table of contents.
toc:
    - name: Intro
    - name: MCTS Background
    - name: Root Parallelism
    - name: Leaf Parallelism
    - name: Tree Parallelism
    - name: Scalable Distributed MCTS
    - name: Watch the Unobserved in UCT (WU-MCTS)
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

Recently, there has been a large focus on training Large Language Models (LLMs) with up to trillions of parameters, using vast amounts of compute for training. However, expecting these models to produce perfect answers instantaneously -- especially for complex queries -- can seem impractical. Naturally, the AI industry is shifting from simply scaling model size to optimizing test-time compute. This shift involves finding ways to harness computational resources effectively during inference. One promising approach is leveraging search algorithms, which enable models to plan, reason, and iteratively refine their outputs. If we can use search in a scalable way, we can leverage vast compute resources to tackle complex challenges such as proving the Riemann Hypothesis or discovering drugs for rare diseases. In this blog post, we will focus our discussion on one such method for leveraging test-time compute: **Monte Carlo Tree Search (MCTS)**.

Rich Sutton’s _bitter lesson_ encapsulates a key insight that is highly relevant to this shift in focus:

> “One thing that should be learned from the bitter lesson is the great power of general-purpose methods, of methods that continue to scale with increased computation even as the available computation becomes very great. The two methods that seem to scale arbitrarily in this way are search and learning.”

This lesson underscores the importance of scalable methods like search, which can capitalize on increased computational power to deliver more robust results. MCTS algorithms have proven successful in many domains having large state spaces (e.g. Go, chess, protein folding, molecular design). However, it is difficult to parallelize MCTS without degrading its performance, since each iteration requires information from all previous iterations to provide an effective exploration-exploitation tradeoff <d-cite key="liu2020watch"></d-cite>. In this blogpost, we will explain and analyze different methods for effectively scaling MCTS.

## MCTS Background

Monte Carlo Tree Search (MCTS) is a powerful algorithm for decision-making in large state spaces, commonly used in games, optimization problems, and real-world domains such as protein folding and molecular design. MCTS stands out for its ability to search complex spaces without the need for additional heuristic knowledge, making it adaptable across a variety of problems. Before MCTS became prominent, techniques like minimax with alpha-beta pruning were the standard in game AI. While alpha-beta pruning could efficiently reduce the search space, its effectiveness often depended on the quality of evaluation functions and move ordering. MCTS offered a different approach that could work without domain-specific knowledge, though both methods can benefit from incorporating heuristics <d-cite key="swiechowski2022mcts"></d-cite>.

### Vanilla MCTS

MCTS iteratively builds a search tree, collecting statistics on potential actions to make intelligent decisions. The algorithm operates in four main phases:

1. **Selection**: Starting from the root node, MCTS selects child nodes according to a policy that balances exploration and exploitation. This continues until either a node with unexplored children is reached (where expansion is possible) or a terminal node is reached.
2. **Expansion**: If the selected node is not a terminal, the algorithm chooses an unexplored child node randomly and adds it to the tree.
3. **Simulation**: From the newly expanded node, a simulation (or "rollout") is conducted. This is typically done by playing out the game or continuing through the state space until a terminal state is reached, using a default policy (often random).
4. **Backpropagation**: Once a terminal state is reached, the result of the simulation is propagated back up the tree, updating the statistics of the visited nodes to reflect the outcome.

MCTS is an **anytime** algorithm, meaning it can be stopped at any point during its execution and still return the best decision found up to that point. This is particularly useful when dealing with problems where the state space (e.g., a game tree) is too large to be fully explored. In practical applications, MCTS operates within a computational budget, which could be defined by a fixed number of iterations or a set amount of time.

The decision-making process in MCTS can be represented mathematically. At any point during the search, MCTS recommends the best action $a^*$ based on the current statistics:

$$
a^* = \underset{a \in A(s)}{\operatorname{argmax}} Q(s, a) \tag{1}
$$

where:

-   $A(s)$ represents the set of actions available in state $s$.
-   $Q(s, a)$ denotes the empirical average result of selecting action $a$ in state $s$.

This equation implies that as more iterations are completed, the statistics for each action become more robust, increasing confidence that the suggested action \( a^\* \) approaches the optimal solution.

### The UCT Selection Policy

A key component of MCTS is the **Upper Confidence Bounds for Trees (UCT)** algorithm, introduced by Kocsis and Szepesvári <d-cite key="kocsis2006bandit"></d-cite>. It determines how child nodes are selected during the selection phase. There are two cases. (1) If a given node has not expanded all of its leaf nodes, then we expand them randomly. (2) Otherwise we select the node with the highest UCT value. The aim of the selection policy is to maintain a proper balance between the exploration (of not well-tested actions) and exploitation (of the best actions identifed so far) <d-cite key="swiechowski2022mcts"></d-cite>. The UCT formula is given by:

$$
a^* = \underset{a \in A(s)}{\operatorname{argmax}} \left\{ Q(s, a) + C \sqrt{\frac{\ln N(s)}{N(s, a)}} \right\} \tag{2}
$$

where:

-   $a^*$ is the action selected from state $s$.
-   $A(s)$ is the set of actions available in state $s$.
-   $Q(s, a)$ represents the average result of playing action $a$ in state $s$ based on simulations performed so far.
-   $N(s)$ is the number of times state $s$ has been visited.
-   $N(s, a)$ is the number of times action $a$ has been played from state $s$.
-   $C$ is a constant controlling the balance between exploration and exploitation. In general, it is a domain-dependent parameter.

We will now move on to explaining an extension to vanilla MCTS and then dive in to the different ways to implement a scalable distributed MCTS solution. If you would like to learn more about Vanilla MCTS, I would highly recommend [this blogpost](https://jeffbradberry.com/posts/2015/09/intro-to-monte-carlo-tree-search/ "Introduction to Monte Carlo Tree Search") on MCTS by Jeff Bradberry.

### Deep RL + MCTS

Since part of the popularity of MCTS is due to its use in the AlphaGo line of work (AlphaGo, AlphaZero, MuZero, etc.), we wanted to expand on how MCTS is incoporated into this work. We will focus on AlphaZero and MuZero, explaining what they are and how they are different. AlphaZero uses a general-purpose MCTS algorithm and directly uses the state transition function for the game of Go. AlphaZero doesn't just use the policy network to choose actions, which could cause spurious approximation errors. Instead, it combines its policy network with MCTS, effectively improving the policy and averaging over approximation errors <d-cite key="schrittwieser2020mastering"></d-cite>. After conducting search with MCTS starting from the current state, AlphaZero returns a vector representing the probability distribution over the next moves. Additionally, the policy network is trained to maximize the similarity between its policy vector and the search probabilities derived from MCTS via the cross entropy loss. A very important detail is that both methods use **virtual loss** to evaluate the N most promising positions in parallel, a form of **tree parallelism** that we will describe in more detail later.

MuZero further generalizes this approach, incorporating a learned state transition function for planning in environments with single-agent domains and intermediate rewards, such as Atari games. MuZero matched the performance of AlphaZero in games of Go, chess, and shogi even though MuZero had no knowledge of the game rules. They also show the scalability of planning in Go and Atari games:

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/2025-04-28-scalable-mcts/mu_zero_scaling_1.png" class="img-fluid-2 rounded z-depth-1" %}
    </div>
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/2025-04-28-scalable-mcts/mu_zero_scaling_2.png" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
  Evaluations of MuZero on Go (A), and all 57 Atari Games (B) <d-cite key="schrittwieser2020mastering"></d-cite>
</div>

Here we see that MuZero's performance in Atari games improved as the amount of search increased, but the improvements were not as significant for Atari as they were for Go. The authors believe this is likely because of greater model inaccuracy in Atari games <d-cite key="schrittwieser2020mastering"></d-cite>, which suggests performance gains can be attained by improving the learned generative models.

### More Extensions to Vanilla MCTS

While we have discussed the vanilla Monte Carlo Tree Search (MCTS) algorithm, there are numerous modifications which enhance its flexibility and applicability across different domains. These modifications adapt MCTS for a variety of complex scenarios including games with both perfect and imperfect information, and extend its utility to real-world applications such as planning, security, and chemical synthesis. For those interested in a deeper exploration of these extensions, a detailed overview can be found in the comprehensive survey by Swiechowski et al.<d-cite key="swiechowski2022mcts"></d-cite>

## How can we scale it?

If we are constrained by time, we can find ways to improve our MCTS by looking at how we can distribute the four different phases of MCTS. There are 3 main ways that MCTS can be parallelized, and we will discuss each one below.

### Root Parallelism

Root parallelism involves creating multiple independent instances of the MCTS algorithm, each building a separate tree concurrently. This approach is straightforward and avoids the complexities associated with managing shared memory. It is somewhat akin to ensemble methods in machine learning where multiple models (trees, in this case) vote on the best action. Each tree explores different paths independently, which can provide a broader exploration of the state space but may result in redundant computations.

While root parallelism is easy to implement and scales linearly with the number of processors, it often lacks efficiency because the separate trees do not share insights. This can lead to a situation where all instances explore the same or similar states without leveraging the knowledge gained by others. To mitigate this, some implementations aggregate the results of different trees, selecting the most promising moves based on a majority vote or averaging method.

### Leaf Parallelism

Leaf parallelism focuses on the simulation phase of MCTS. By distributing the rollout simulations across multiple workers, it achieves a high degree of parallelism with minimal coordination overhead.

Although leaf parallelism increases the speed of statistical accumulation, its effectiveness can be limited by the quality of rollouts. If early simulations indicate a poor choice, continuing to explore that choice can waste computational resources. Adaptive strategies that adjust based on intermediate results can help optimize this approach.

### Tree Parallelism

Tree parallelism, unlike root parallelism, involves multiple threads or processes working on the same tree. It aims to expand the tree more quickly by allowing concurrent operations on different parts of the tree.

The main challenge with tree parallelism is avoiding conflicts and ensuring data integrity as multiple threads update the tree. Techniques such as lock-free programming and the use of virtual loss have been developed to manage these issues. Virtual loss temporarily penalizes nodes being explored by one thread, discouraging other threads from exploring the same nodes simultaneously.

## Scalable Distributed MCTS

In 2011, Yoshizoe et al <d-cite key="yoshizoe2011scalable"></d-cite> introduced an efficient scheme for parallelized MCTS with two core ideas: transposition-table driven scheduling (TDS) parallelism and depth-first MCTS. First, we will discuss these topics separately and then see how the combination of the two concepts leads to efficient parallelization.

First, we introduce TDS. At a high level, TDS is a mechanism for evenly distributing data across W worker nodes and efficiently distributing work to be done on said data. In TDS, each record of data is passed through a hash function that maps it to one of the worker nodes. This is the record’s “home”: the record is stored in memory on its home worker and on no other workers. With a good hash function, this scheme ensures that the data is partitioned evenly across a network of workers. The key idea behind TDS is that data doesn’t move, requests do. If worker A receives a request to run some function on a partition of data that resides on worker B, the request is forwarded to worker B and the response is computed locally before being sent back to worker A. This design is very efficient, as moving data across a network is far slower than encoding requests to process that data and sending the requests where the data resides.

An important observation to make is that TDS communication is asynchronous. When worker A receives the request to process data that actually resides on worker B, all it has to do is forward the request from B. Worker A does not need to pause local computation and wait for worker B’s response before handling other requests. Thus, even though communication is frequent in this scheme, its cost is offset by its inherent asynchronicity.
Before thinking about how TDS could support parallel MCTS, let’s understand how it could support sequential MCTS. Yoshizoe et al hash each node in the MC tree to a unique worker. Each time a new node is discovered via the MCTS search process, its home worker is discovered and metadata about the node is stored there: its parent, its children, its value, and its visit count.

In addition to outlining the procedure to create a new node, the authors describe the two types of messages sent between workers. _Report(N,R)_ is used when backpropagating the result of a rollout. It instructs node N to increment its UCB count and adjust its value given the result of the rollout R. _Search(N)_ instructs a worker to either expand the tree if N is a leaf by continuing a random rollout locally or to forward the search job to the child of node N determined via UCB if N is not a leaf. It should make sense that after _Search(N)_ is called on a leaf node, it must call _Report(Parent(N))_ to start the result propagation procedure, which continues until the root is reached.

Each worker runs a very simple continuous while loop where it (1) checks if a new job has arrived and appends it to a job queue if one has and (2) processes a request from its job queue. To ask the system to perform 100 rollouts, we would call _Search(Home(root))_ where Home() is the hash function that maps a state/node to its home worker node. While this procedure is not as precise as sequential MCTS because some nodes could reuse “stale” UCB statistics on multiple _Search()_ calls before updating with the corresponding _Report()_ calls, the authors suggest that the algorithm is a close approximation to sequential UCB and their empirical results (on both performance and especially computation speed enabled by parallelism) support their claim.

Before moving forward, we must address a major flaw in this paradigm. As the number of nodes grows, the number of requests sent across the system grows accordingly. However, even though the nodes are roughly evenly distributed across the workers via the TDS hashing protocol, the work done on each worker node is not evenly distributed. Workers that house nodes “higher up” in the search tree which are visited more frequently process far more jobs than workers that do not have such nodes. This problem is especially clear for the worker that houses the root node, which must process requests for every single rollout handled by the system.

In an attempt to mitigate the uneven distribution of job requests, the authors propose using a depth-first version of MCTS. The core innovation found in df-MCTS is that the frequency of backpropagation steps (i.e., the frequency of _Report()_ messages) is greatly reduced compared to normal MCTS. df-MCTS acts like normal MCTS when it receives a _Search()_ job, but it doesn’t always report results to its parent when a _Report()_ message is received. Instead, it either sends a new _Search()_ message to one of its children if more exploration is needed or, if the node has been thoroughly explored, it sends a _Report()_ message to its parent. The logic behind the decision to send a _Search()_ or a _Report()_ in response to a child’s _Report()_ is not fleshed out by the authors and is implementation dependent.
The authors tested their system against standard, sequential MCTS on the same 2-player benchmark game. They found that if both algorithms trained for the same amount of rollouts, sequential MCTS slightly outperforms df-MCTS. This is expected, as the delayed propagation of rewards under df-MCTS means that some exploration decisions are made based on stale UCB statistics. However, when both algorithms were allowed to train for the same amount of wall-clock time, the distributed system significantly outperformed sequential MCTS as it was able to perform substantially more rollouts in the same time period.

The final system presented by the authors enables parallelized MCTS by (1) evenly distributing the search tree and its metadata across W workers under a partitioning based on TDS, (2) performing all work on the partitioned tree locally without sending data across the network, and (3) better balancing the work done on each worker node by running a depth-first version of MCTS which takes backpropagation steps less frequently and puts less strain on the worker housing the root node. The distributed MCTS system significantly outperformed sequential MCTS given the same amount of train time in the author's experiments, suggesting it is a viable approach to parallelizing MCTS.

## Watch the Unobserved in UCT (WU-MCTS)

Recall that in the standard MCTS procedure each node in the explored tree stores a scalar called the “value” that encapsulates how advantageous we think it is to visit that node based on our previous experience. Using these values, we can characterize our “best” possible rollout as greedily choosing the child with the highest value when we perform rollouts, which take us from the root down to a leaf node in our known tree. It is crucial to remember that always choosing the “best” possible rollout fails to adequately balance exploiting our current knowledge against the potential of acquiring novel expertise via exploring. In vanilla MCTS, this balance is struck with UCB (Upper Confidence Bound). Under UCB, each node stores a count of how many times it has been visited in previous rollouts, and these counts are used to incentivize exploration as we showed in the formulation of the UCB selection policy earlier (see Equation 2).

Each time a rollout finishes, a recursive procedure propagates the result/reward of the rollout back up the tree. Each node visited during the rollout in the known tree increases its stored visit count by one and updates its value based on the result of the rollout. The next rollout will use the updated visit counts and values. Based on this formulation, it is impossible to efficiently parallelize MCTS with UCB: if we naively try to parallelize the rollouts by having W workers run W rollouts based on the same values and visit counts, they will all use the same path within the known tree and we will effectively have no exploration.

Liu et al [paper citation] recognized this problem and proposed an elegant solution to enable parallelized approximate MCTS. Their simple fix involves tracking an additional number $O_s$ at each node which counts the number of active rollouts which visited that node. Then, the update rule becomes:

$$
a_s = \underset{s' \in C(s)}{\operatorname{argmax}} \left\{ V_{s'} + \beta \sqrt{\frac{2 \log (N_s + O_s)}{N_{s'} + O_{s'}}} \right\} \tag{3}
$$

This keeps the UCB decision rule but allows for multiple rollouts to be active at the same time by using the sum of completed rollouts and active rollouts to regulate exploration at each node.

The system proposed by Liu et al has many moving parts but is straightforward to understand. The authors define three types of workers: the master worker, expansion workers, and simulation workers. The master worker, as expected, runs the main MCTS logic and stores the truth version of all global data structures (i.e., nodes in the tree/tree structure plus values and visit counts for each node). It runs until it has successfully completed a predetermined number of rollouts. Crucially, each rollout can be broken down into two distinct phases: (1) the expansion phase in which the known tree is traversed using UCB until a leaf is reached and (2) the simulation phase, which starts at a leaf in the known tree and continues to a terminal state making random traversal decisions.

Expansion workers take in a tree state and run UCB until a leaf is reached. They then return the path taken to the master and the new leaf is placed in the simulation buffer. The master then makes an incomplete update, in which it increments the active rollout counter for each node on the path it received from the expansion node. Meanwhile, simulation nodes take in a state (represented by a leaf chosen from the simulation buffer) and randomly play until a terminal state is reached. The simulation nodes then return the reward achieved during the random simulation to the master.

The master node finally back propagates this return and updates the visit counts (increment) and active rollout counts (decrement) for corresponding tree nodes. This algorithm can be best understood by tracing the below block diagram, which was included in the original paper:

{% include figure.html path="assets/img/2025-04-28-scalable-mcts/wu_mcts_diagram.png" class="img-fluid" %}

Understanding what data is communicated between nodes and when the data is sent is crucial to understanding how this method scales with more workers and larger trees.

Each time an expansion worker receives a task from the master, it receives a complete copy of the tree state (so this message grows over time). Upon completion, the expansion node returns the new leaf node, the reward it received when reaching the new leaf, and the path taken to get there. The simulation workers require much smaller messages: the master tells them to start simulating from some state and they return the reward accumulated during their simulation.

While there are many rounds of communication, the authors showed that the expansion and simulation steps took longer than communication in their system, suggesting communication is not a bottleneck.

To test their procedure, the authors benchmarked against sequential MCTS and found only slight performance degradation on an RL benchmark suite of 15 Atari games. This method also outperformed existing parallel MCTS algorithms on the benchmark suite, beating leaf and root parallelization on 15/15 games and tree parallelization on 13/15.

In short, WU-UCT breaks down the MCTS process into two distinct phases (expansion and simulation) which can be parallelized by implementing an adjusted UCB algorithm that tracks the number of rollouts that are active in addition to the number of times each node in the tree has been visited. The empirical results of the study demonstrate that this is an effective MCTS parallelization method, with only limited performance setback vs sequential MCTS and a strong showing against previous parallelized MCTS methods.

## Conclusion