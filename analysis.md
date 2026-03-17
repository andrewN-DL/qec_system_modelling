### Method
I developed a discrete event simulation model of the system. The simulation advances time directly between events. The main events considered are 
- Qubit arrivals into the A queue
- Components A, B, C completeing qubit processing

Qubit arrivals are modelled as a Poisson process with mean arrival time $\lambda$. Qubits enter and leave the queues by increasing or decreasing the length property of the respective queues. I do not model the FIFO nature of the queues as we are not concerned with ordering of specific qubits, only higher level system/component behaviour.

Qubits can enter a particular component if:
- There are qubits in its queue
- The component is not busy
- It is not blocked by queues in front of it

Once a component cas completed processing, each processed qubit displaces a previously held qubit into the next queue. If the next queue is at capacity, the component becomes blocked and cannot process further qubits until space becomes available.

System state is updated only at event times. We can easily collect a wide range of data as the simulation runs, allowing us gather overall system and component statistics

### Asumptions
1. Queues are unordered. Focus on throughput as opposed to specific information encoded.
2. Systems and queues initially empty. Max throughputs/utilistaions may be slightly higher due to this
3. Time to travel between components is zero

### Validation
I validated the system using conservation of qubits to ensure all qubits were accoubnted for. Need to have

```python
qubits_in = qubits_out
            qubits_in_queues + # qubits in each queue
            qubits_in_components + # Max one in each component
```

We also expect to have a decoding rate ~ $1/\lambda$ for largrer values of lambda as for stable queues qubits in = qubits out

### Results & Analysis

The simulation can be used with different input parameters to to investigate how these different parameters might change the system. As the sim in not deterministic, multiple runs are performed for each value of $\lambda$ to obtain averaged results and estimate variability.


#### Throughput (Decoding Rate)


![Decoding rate](/plots/decoding_rate.png)
The maximum acheivable decoding rate for the system is approximately 0.47qubits/ $\mu s$. Lower values of $\lambda < 2.1$ operate at this decoding rate, limited by the slowest component (component B). As $\lambda$ increases to $2.1 - 2.2$, the queue at B spends less time at capacity, and this rate starts to drop. 

As $\lambda$ continues above 2.5, A, and B queue lengths continue to fall until they are empty for the majority of the time. Qubits entering the system do not need to spend much time queueing, or for larger values of $\lambda$, they spend no time at all. Decoding rate becomes ~ $1/\lambda$, as the lack of backlogs means there is a stable flow of qubits through the system.




#### Latency

![Latency](/plots/total_latency.png)
![Latency entering A](/plots/latency_entering_a.png)

Latency is defined as the total time qubits spend in the system
- For low values of $\lambda$ latency is high. As the simulation proresses, the latency continues to increase as the bottleneck at component B causes queues at A to continuiously increase. Changing $\lambda$ around this range only the rate at which these queues/latency changes, but does not stabilise it
- For $\lambda = 2.1-2.3$ the latency stabilises, reaching values of approximately $20\mu s$. The queues at A are no longer growing infinitely
- As $\lambda \rightarrow \infty$ we expect average queue lengths to go towards zero, and latency drops to the sum of the means of the component processing times.

We can also examine the time spent in the system from entering A. For values of $\lambda < 2$ it is much lower than the total latency, showing that the extra time overall spent in the system is due to tike spent in A queues
 
#### Component Utilisation
![Utilisation](/plots/component_utilisation.png)

Component utilisation is the portion of time each component spends processing qubits. For all values of $\lambda$, component B has thr highest utilisation, followed by component A. Component C has a utilisation significantly lower. This is further indication that component B is the bottleneck, and component C can clear its queues quickly, spending more time idle.

- For lower values of $\lambda$ utilisation of B sits cloe to 1. It is working at maximum capacit, causing quubits to pile up behind.
- At the critical value of $\lambda$ all utilisations start to decrease. Component B starts to clear its queue at times and spends some time idle. This decreases qubits put into queue C. This decrease, as well as generally less qubits arriving also decreases utilisation of Component A.
- As $\lambda$ increases utilisation scalse like $1/\lambda$. Components spend less time processing as queues are cleared and less qubits arrive.



#### Dynamics & Queue Lengths

![Queue Lengths](/plots/average_queue_lengths.png)
![Stopped Times](/plots/components_stopped_times.png)

As discussed above, low values of $\lambda$ generally overloads the system. More qubits arrive than can be processed by the system. In particular, component B is the bottlenect. Its queue is constantly near capacity causing component A to be blocked and qubits to pile up in queue A.

We approach critical behaviour of the system at approximately $\lambda = 2.1-2.3$. This is where the B queue stops maintaining a queue length clost to capacity. 

As $\lambda$ continues increasing above 2.5, the system becomes underutilised. All components easily clear their queues and spend more time idle. Decoding rate drops as qubits arrive less frequently and components no longer get blocked by full queues downstream

#### Queue lengths over time for different values of $\lambda$
![Lam = 0.2](/plots/queue_0.2.png)
![Lam = 2.15](/plots/queue_2.15.png)
![Lam = 4](/plots/queue_4.png)

The latency for low values of lambda is very high. This reflects the long times qubits spend in queues at A as it grows infinitely. For constant, low values of $\lambda$, total latency will not stabilise, as qubits wait longer as the A queue length increases. increasing the value of $\lambda$ decreases the rate at which this queue grows, but does not allow the queue to stabilise until $\lambda = 2.1$. We can also examine times as from when qubits enter component A. For low values of $\lambda$ there is a constant time in the system of ~$170\mu s$. This reflects times when the B queue is at capacity. As long as the B queue is operating close to capacity, changes in $\lambda$ will have no effect of most high level statistics (utilisations, decoding rate, etc), instead the only factor that changes is the rate of growth of the A queue



As we increase $\lambda$ to approximately $2.1 - 2.2$ we reach a point where the queues at B start to stabilise, as opposed to staying at capacity. Average queue length for B drops dramatically. The portion of time that component A spends blocked drops, helping the A queue to stabilise. This is the ideal maximum decoding rate at ~ 0.47qubits/ $\mu s$. Latencies drop dramatically to ~ $15\mu s$ as queues shorten. Although queues are not infinitely growing, we still see B is the bottleneck at these values of lambda, as the queue lengths for C remain very low.



As $\lambda$ continues above 2.5, A, and B queue lengths continue to fall until they are empty for the majority of the time. Qubits entering the system do not need to spend much time queueing, or for larger values of $\lambda$, they spend no time at all. Decoding rate becomes ~ $1/\lambda$, as the lack of backlogs means there is a stable flow of qubits through the system. Utilisation for all components drops as the system/components regulary completely clear themselves of qubits before new ones enter. As $\lambda \rightarrow \infty$ we expect average queue lengths to go towards zero, and latency drops to the sum of the means of the component processing times.

