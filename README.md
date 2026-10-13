# Readout Pipeline Queueing Simulation

Discrete-event simulation of a three-stage processing pipeline for qubit readout data (A → B → C) with finite buffers and backpressure, used to find the bottleneck stage and the maximum sustainable arrival rate.

## Overview

**Motivation.** In a fault-tolerant quantum computer, syndrome and readout data must be processed in real time. If the classical processing pipeline cannot keep pace with the arrival rate, a backlog builds and latency grows. This project models a simplified multistage pipeline to find its maximum sustainable rate and its bottleneck. It is a simplified model, not a model of a specific hardware system.

The simulation analyses:

- **Throughput:** the number of readout messages processed per microsecond.
- **Latency:** the time taken for a message to travel through the system and produce an output from Stage C.
- **Component utilisation:** the proportion of time each component spends processing messages.
- **Queue dynamics:** how queue lengths and component blocking change with the input arrival rate.

The model is implemented in Python using a discrete-event simulation. Multiple runs are performed for each input parameter to estimate average performance and variability.

## System

| Stage | Behaviour | Processing time |
|---|---|---|
| Input | Poisson arrivals, mean inter-arrival time λ; unbounded queue in front of A | n/a |
| A | Reads message; blocks if the A→B buffer is full | Uniform(1.5, 2.2) µs (mean 1.85 µs) |
| A→B buffer | FIFO, capacity 100 | n/a |
| B | Blocks if the B→C buffer is full | Log-normal, μ = 0.5, σ² = 0.5 (mean ≈ 2.12 µs) |
| B→C buffer | FIFO, capacity 25 | n/a |
| C | Output stage | Constant, 1.5 µs |

## Method

### Discrete-event simulation

The simulation advances time directly between events rather than using a fixed timestep. The main events are:

- Message arrivals into the queue for Stage A.
- Processing completions at Stages A, B, and C.

Arrivals are modelled as a Poisson process with exponentially distributed inter-arrival times. The parameter $\lambda$ is the **mean inter-arrival time**, in microseconds, so the mean arrival rate is $1/\lambda$ messages per microsecond.

Each component processes one message at a time, with processing times sampled from the distributions in the table above.

Queues are represented by their lengths rather than by storing individual messages. This is sufficient for analysing aggregate system performance.

### Blocking and component interactions

The system uses a knock-on mechanism: after a component has been primed, processing a new message displaces a previously held message into the downstream queue.

If a downstream queue reaches capacity, the upstream component becomes blocked and cannot continue processing until space becomes available. This models backpressure and allows congestion to propagate upstream.

### Data collection

The simulation records, for each run:

- Queue lengths over time.
- Component processing time and utilisation.
- Component blocking time.
- Message arrival and output times.
- Mean latency and individual observed latencies.
- Overall throughput.
- Final queue lengths.

Multiple runs are performed for each value of $\lambda$ and the results are averaged.

## Assumptions

1. **Unordered queues:** Only queue lengths are tracked. FIFO ordering is not explicitly modelled because the analysis focuses on aggregate performance.
2. **Initially empty system:** All queues and components start empty, which introduces an initial transient that may affect measured performance.
3. **Zero transfer time:** Travel time between components is negligible.
4. **Single-server components:** Each component processes at most one message at a time.
5. **Stochastic arrivals and service times:** Arrivals and processing times follow the distributions defined above.

## Validation

**Conservation check.** At any time, the number of messages that have entered the system should equal the number that have exited plus the number still present in queues or components:

$$
N_{\mathrm{in}} =
N_{\mathrm{out}} +
N_{\mathrm{queues}} +
N_{\mathrm{components}}
$$

The component count includes messages currently held or being processed at each stage, with at most one held message per component.

**Limiting behaviour.** For sufficiently large $\lambda$, arrivals are infrequent and the system is input-limited, so throughput should approach $1/\lambda$. At higher arrival rates, throughput is limited by the processing capacity of the system.

**Analytic check.** The mean service time of Stage B is $\exp(\mu + \sigma^2/2) \approx 2.117\,\mu s$, giving a maximum throughput of approximately $0.472$ messages per microsecond. The other stages are faster (Stage A: $1/1.85 \approx 0.54$; Stage C: $1/1.5 \approx 0.67$), so B is the predicted bottleneck. This matches the simulated plateau of approximately $0.47$ and predicts the transition to input-limited operation at $\lambda \approx 2.12\,\mu s$, consistent with the observed range of $2.0$ – $2.2\,\mu s$.

## Results and Analysis

### 1. Queue dynamics and blocking

![Average queue lengths](plots/average_queue_lengths.png)

![Component stopped times](plots/components_stopped_times.png)

For small values of $\lambda$, the system receives messages faster than it can process them. Stage B is the primary bottleneck: its queue remains close to capacity, frequently blocking Stage A and causing messages to accumulate in the A queue.

Stage C generally clears its queue more quickly than Stage B, so its queue remains comparatively small.

As $\lambda$ increases towards approximately $2.0$ – $2.2\,\mu s$, Stage B begins to clear its queue more consistently. Blocking of Stage A decreases, allowing the backlog at A to stabilise.

For larger values of $\lambda$, the system becomes increasingly underutilised. Queues are cleared more frequently, components spend more time idle, and downstream blocking becomes less common.

### 2. Throughput

![Throughput](plots/decoding_rate.png)

The maximum observed throughput is approximately

$$
0.47\ \text{messages}/\mu s,
$$

which matches the analytic bound $1/\mathbb{E}[S_B] \approx 0.472$ derived above.

For small $\lambda$, the system operates near this limit. Additional arrivals cannot increase the output rate because Stage B limits the processing capacity. Instead, excess arrivals accumulate in the upstream queue.

As $\lambda$ increases towards approximately $2.0$ – $2.3\,\mu s$, throughput begins to decrease as the system transitions towards an input-limited regime.

For $\lambda$ above approximately $2.5\,\mu s$, queues are generally short and throughput approaches

$$
R \approx \frac{1}{\lambda}.
$$

In this regime, throughput is determined by how frequently messages arrive rather than by the processing capacity of the components.

### 3. Latency

![Total latency](plots/total_latency.png)

![Latency measured from entry into Stage A](plots/latency_entering_a.png)

Latency is the elapsed time between a message entering the system and its output being produced by Stage C.

For small $\lambda$, latency is high and can increase throughout the simulation. The input rate exceeds the sustainable processing rate, so the queue at A grows and later arrivals wait longer.

Between approximately $2.0$ and $2.5\,\mu s$, latency decreases substantially and stabilises at around $20\,\mu s$ in the observed results. This corresponds to reduced queueing delays as Stage B becomes less congested and Stage A is blocked less frequently.

At larger values of $\lambda$, queueing delays become small. As arrivals become infrequent, the mean latency approaches the total mean processing time across the three stages ($1.85 + 2.12 + 1.5 \approx 5.5\,\mu s$), assuming the system is otherwise empty

Latency measured from entry into Stage A excludes the time spent waiting in the initial A queue. Comparing the two measures shows how much of the total latency is due to upstream queueing, which is particularly significant under heavy load, where the A queue dominates the waiting time.

### 4. Component utilisation

![Component utilisation](plots/component_utilisation.png)

Component utilisation is the proportion of simulated time that a component spends processing messages.

Across the tested values of $\lambda$, Stage B has the highest utilisation, followed by Stage A. Stage C has significantly lower utilisation, indicating that it generally has sufficient capacity for the messages it receives.

For small $\lambda$, Stage B operates close to full utilisation, and its limited capacity causes upstream congestion and blocking.

Near the transition to stable operation, Stage B begins to clear its queue intermittently and spends more time idle. Utilisation across the system decreases as the arrival rate falls.

For large $\lambda$, utilisation decreases further because components frequently finish processing before additional messages arrive.

### 5. Queue lengths over time

The following plots illustrate the different operating regimes.

#### High arrival rate: $\lambda = 0.2\,\mu s$

![Queue lengths for lambda 0.2](plots/queue_0-2.png)

The arrival rate is high relative to the system's processing capacity. Stage B remains congested, and the queue at A accumulates a backlog.

#### Near the stability threshold: $\lambda = 2.15\,\mu s$

![Queue lengths for lambda 2.15](plots/queue_2-15.png)

The system operates near its capacity limit. Stage B is less persistently congested, allowing Stage A to process incoming messages without sustained backlog growth.

#### Low arrival rate: $\lambda = 4\,\mu s$

![Queue lengths for lambda 4](plots/queue_4.png)

The system is underloaded. Queues are generally short, and components frequently become idle between messages.

## Conclusion

Stage B is the primary bottleneck in the pipeline. Its mean service time limits the maximum sustainable throughput to approximately $0.47$ messages per microsecond, in agreement with the analytic bound.

When the mean inter-arrival time is too small, the system becomes overloaded: the A queue grows, latency increases, and Stage B remains close to full utilisation. Around $\lambda = 2.0$ – $2.2\,\mu s$, the system approaches the transition to stable operation. At larger values of $\lambda$, queueing delays and utilisation decrease, and throughput becomes limited by the arrival rate.

These results illustrate the trade-off between throughput, latency, and utilisation, and show how simulation can identify bottlenecks and estimate the maximum sustainable input rate of a multistage processing system.
