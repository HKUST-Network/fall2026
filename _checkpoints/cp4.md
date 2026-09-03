---
type: checkpoint
date: 2026-11-09T15:00:00+08:00
tentative: true
title: 'Checkpoint 4: Design your own algorithm!'
due_event: 
    type: due
    date: 2026-11-27T23:59:00+08:00
    description: 'Checkpoint #4 due (tentative)'
---

# Introduction
In this checkpoint, you need to finish two tasks based on **your own CP3 code**. So if you have not finished your CP3, you need to implement it at first (**we don't provide sample answer code**). We will add some regression tests for CP3 in CP4.

* Dr. Matt Mathis's hypothesis,
* Your own CCA.

# Dr. Matt Mathis's hypothesis
## Hypothesis
In Checkpoint 1, we asked you questions and you formed your own hypotheses. In this checkpoint, we are going to give you a hypothesis formulated by Dr. Matt Mathis: that TCP Reno implementations obey the following throughput equation:
**throughput = (MSS/RTT)*(C/sqrt(p))**, where **C** is a constant, **p** is the loss probability, RTT is the round-trip time and MSS is 1400 bytes in our project. The following measurements will help you verify this hypothesis.

(Reference: DOI 10.1145/263932.264023) <!-- https://dl.acm.org/doi/10.1145/263932.264023 -->
## Experiment
Use tcconfig (checkpoint 1) to adjust the loss probability **p** on the sender node to different values.
```
sudo tcset eth0 --rate 10Mbps --delay 20ms --loss 0.01%
```
For different values of **p**, **transfer a large file** and measure the duration of the transfer (make sure to adjust the file size so that your transfer takes at least a few seconds). Use the transfer duration and the file size to calculate the throughput. Repeat the same experiment 10 times for each loss probability. Create a line plot with **1/sqrt(p)** on the x axis and throughput on the y axis. Do linear regression and include the regression line on the same plot.

## Inference
Based on the results of your experiments, we ask that you answer the following questions:
* Using your linear regression, what value did you find for **C**?
* Compute the Pearson Correlation Coefficient between **1/sqrt(p)**  and the throughput that you measured.
* Does your data corroborate Dr. Mathis's hypothesis? Why or why not?

## Hint
Due to the lack of the timeout retransmission, sometimes your program may stuck please try again.

# Your own CCA
You now have the opportunity to design your own CCA with the goal of outperforming TCP Reno and you can modify anything you like.
* The metric is throughput (as measured by the time it takes to transfer a 1MB file).

## Hint
* You can go through the lecture: Congestion Control and lecture: Advanced Congestion Control again.
* Due to the lack of the timeout retransmission, sometimes your program may stuck please try/submit again.
* <font color=red> <strong >Test link: RTT = 200ms, the buffer size of the router is infinite. But the router will add extra delay Xms when the buffer depth exceeds N. </strong> </font> For example, if N = 5 and there are currently 10 packets in the buffer, extra delay will be added for packets 6 through 10.

## How to test by yourself
### tcconfig
You can simulate the link (bandwidth, latency and loss rate) by tcconfig.
```
sudo tcset eth0 --rate 10Mbps --delay 20ms --loss 0.01%
```
### burst
You can simulate the burst traffic by starting an another file transfer between two VMs.
### duplicate ACKs
You can manually send duplicate ACKs at your recevier to simulate the congestion scenarios.

# What to submit
Like the checkpoint 1, we have one assignment for the codes and one assignment for the report.
## Codes
Your implementation should be in the following files: ```foggy_function.cc```, ```foggy_tcp.cc```, ```foggy_function.h``` and ```foggy_tcp.h```. You can run the `submit.py` at the root of the project which will generate a `submit.zip` file including these four files, then submit the `submit.zip` file to Gradescope. The autograder will copy those four files you submitted to the starter codes and do the testing.

We will evaluate your CCA by autograder and use a leaderboard to show the results in real time.
## Report
The report should include two parts: (1) Dr. Matt Mathis's hypothesis and (2) Your own CCA.
### Dr. Matt Mathis's hypothesis
* Create a line plot with **1/sqrt(p)** on the x axis and throughput on the y axis.
* Do linear regression and include the regression line on the same plot.
* Using your linear regression, what value did you find for **C**?
* Compute the Pearson Correlation Coefficient between **1/sqrt(p)**  and the throughput that you measured.
* Does your data corroborate Dr. Mathis's hypothesis? Why or why not?

### Your own CCA
In your report include a section called **Algorithm Proposal**. In this section, you should:
* Propose a new algorithm (or a modification to Reno) that improves its throughput. Provide a detailed algorithm description, including why your new algorithm will improve throughput relative to your Reno implementation.

In your report include a section called **Algorithm Evaluation**. In this section, you should:
* Provide data comparing your Reno implementation to your new algorithm completing a 1MB transfer.
* That means how long does it take for a single connection to transfer 1MB.

<!-- * Provide data comparing your Reno implementation to your new algorithm completing a 20MB transfer. Compare the following properties of both algorithms:
    1. How long does it take for a single connection to transfer 20MB?
    2. How long does it take for two connections, sharing the same link, to transfer 20MB each? 
    2. What is the Jain's Fairness Index when the two connections compete?

We will not rank your result by the fairness and we just want you to have a basic sense of fairness (e.g., how to evaluate fairness). So we will accept any reasonable result.

Jain's Fairness Index (JFI) is a measure of how fairly a set of entities share some resource: $$ J(x_1, x_2, \ldots, x_n) = \frac{(\sum_{i=1}^{n}{x_i})^2}{n \cdot \sum_{i=i}^{n}{x_i^2}} $$, where $$x_i$$ is the resource allocation assigned to entity $i$ (e.g., throughput). JFI values range from $$\frac{1}{n}$$ (very unfair) to 1 (perfectly fair). -->