---
type: checkpoint
date: 2026-09-07T15:00:00+08:00
tentative: true
title: 'Checkpoint 1: Kickoff'
# link: https://www.gradescope.com/
due_event: 
    type: due
    date: 2026-09-25T23:59:00+08:00
    description: 'Checkpoint #1 due (tentative)'
---

# Validate your GitHub student status

Please do it as soon as possible. It may take a few days to validate your status. Then you can use Copilot for free. You may refer to this guide <!-- [this guide](https://docs.github.com/en/copilot/managing-copilot/managing-copilot-as-an-individual-subscriber/managing-your-copilot-subscription/getting-free-access-to-copilot-as-a-student-teacher-or-maintainer) -->.

# Create a **PRIVATE** repo of your project

Clone the repo to your local machine.
```
git clone https://github.com/HKUST-Network/foggytcp.git
```
*Reminder: Use git to clone the repo instead of download as a zip. Otherwise you cannot track the changes.*

# Install Vagrant and VirtualBox

You can use Vagrant and VirtualBox to automatically setup the environment. Different OSes require different installation steps.

If you are using Windows Subsystem for Linux (Linux), please refer to the installation steps for Linux. I also recommend you to do in that way -- Linux is an important tool for developers.
Otherwise, follow the instructions below.

* Install Vagrant <!-- [Install Vagrant](https://www.vagrantup.com/downloads.html) -->
* Install VirtualBox <!-- [Install VirtualBox](https://www.virtualbox.org/wiki/Downloads) -->

*FAQ: Where is the terminal on my laptop?* The best way to access the terminal is in Visual Studio Code. This is the most popular IDE for developers. Under the View menu, you can find the Terminal option. 

# Setup the environment

Once you have both Vagrant and VirtualBox installed, navigate inside this repo and run:

```bash
# In your project root folder where your Vagrantfile is located
vagrant up
# builds the server and client containers using VirtualBox.
```

After the containers are built, you can access the client and server containers using the following commands:
```bash
vagrant ssh client
vagrant ssh server
# connects to either the client or server using SSH.
```
*You will need to open two terminals to connect to server and client separately.*

Vagrant keeps all files synchronized between your host machine and the two containers. 
In other words, the code will update automatically on the containers as you edit it on your computer. 
Similarly, debugging files and other files generated on the containers will automatically appear on your host machine.

## DNS Domain Issue

If your vagrant VMs fail to connect to the Internet and execute commands such as "sudo apt-get install", please use the following command within your VMs:
```bash
sudo bash -c 'echo "nameserver 8.8.8.8" > /etc/resolv.conf'
```

# Run the server and client

## Build the binaries

```bash
# In the server VM
cd /vagrant/foggytcp
mkdir build
make system
```
You will see the binaries `server` and `client` in the `foggytcp` folder.

Remember, each time when you modify the code, you need to rebuild the binaries.

## Read and understand the `server.cc` and `client.cc`

The `server.cc` and `client.cc` are the main files for the server and client respectively.


## Send a file from the client to the server

In the server VM, at the `/vagrant/foggytcp` folder, run the server with the following command:
```bash
./server 10.0.1.1 3120 test.out
```

In the client VM, at the `/vagrant/foggytcp` folder, run the client with the following command:
```bash
./client 10.0.1.1 3120 src/client.cc
```

Now you have successfully transmitted the `client.cc` file from the client to the server, named as `test.out`.


## Set traffic control manually
tcconfig <!-- [tcconfig](https://tcconfig.readthedocs.io/en/latest/index.html) --> can be used to manually set traffic control between client VM and server VM. Use the following command to install it:
```bash
sudo pip install tcconfig
```

Then you need to check the network interface name of your VM:
```bash
ifconfig
```

Then you can set traffic control to this interface. For example, use the following command on the client side to set one-way latency to be 100ms:
```bash
sudo tcset $IFNAME --delay 100ms
```

And use the following command to see your settings:
```bash
sudo tcshow $IFNAME
```

Please refer to the tcconfig documents <!-- [tcconfig documents](https://tcconfig.readthedocs.io/en/latest/index.html) --> for more usages.

# What to submit

## Codes

This checkpoint is about setting up the environment and understanding the code.
You do not need to modify the code for this checkpoint.
We hope that you can familiarize yourself with the submission procedure on Gradescope.

1. Modify any place in the `*.cc` or `*.h` (it can even be adding a new space).
2. Commit your code. If you're using VS Code, follow this video <!-- [this video](https://youtu.be/9cMWR-EGFuY?si=etqYwMOt5sz1QgCL&t=289) --> to commit the changes (from 4m49s to 6m22s).
3. Run the `submit.py` at the root of the project. It will generate a `submit.zip` file.
```
python submit.py
```
4. Submit the `submit.zip` file to Gradescope.

## Report
You need to measure the file transmission time for
* Different file sizes
* Different bandwidths
* Different delays

and plot them in three figures. Remember to fix other two parameters while changing one and make your plots **reader-friendly**. In the report, you need to follow the hypothesis-experiment-conclusion structure: 

* **Hypothesis.** Before running experiments, what is your expected results? Why?
* **Experiment.** Measure the file transmission time under different conditions and plot your results.
* **Conclusion.** Does you experiment results match your expectations? What may cause the gap between them? If your predictions are totally different from your resutls, please hypothesize as to why your predictions were wrong.

The following parameter configurations are for your reference. You are free to test with other configurations to verify your hypothesis.

**Test 1**
- Bandwidth = 10Mbps
- Delay = 10ms
- Filesize = 1KB, 5KB, 25KB, 100KB, 1MB,10MB

To create a file with a desired size:
```bash
fallocate -x -l 1M 1M.txt
```

**Test 2**
- Dealy = 10ms
- Filesize = 1MB
- Bandwidth = 1Mbps, 5Mbps, 10Mbps, 20Mbps

**Test 3**
- Bandwidth = 10Mbps
- Filesize = 1MB
- Delay = 0ms, 5ms, 10ms, 20ms, 50ms, 100ms