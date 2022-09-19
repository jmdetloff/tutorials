**Introduction** \|\| `What is DDP <ddp_theory.html>`__ \|\| `Single-node
Multi-GPU training <ddp_multigpu.html>`__ \|\| `Fault
Tolerance <ddp_fault_tolerance.html>`__ \|\| `Multi-node
training <intermediate/ddp_multinode.html>`__ \|\| `mingpt training <intermediate/ddp_minGPT.html>`__

Distributed Data Parallel in PyTorch - Video Tutorials
======================================================

Authors: `Suraj Subramanian <https://github.com/suraj813>`__

.. raw:: html

   <embed video>

This series of video tutorials walks you through distributed training in
PyTorch via DDP.

The series starts with a simple non-distributed training job, and ends
with deploying a training job across several machines in a cluster.
Along the way, you will also learn about
`torchrun <https://pytorch.org/docs/stable/elastic/run.html>`__ for
fault-tolerant distributed training.

The tutorial assumes a basic familiarity with model training in PyTorch.

Running the code
----------------

You will need multiple CUDA GPUs to run the tutorial code. Typically
this can be done on a cloud instance with multiple GPUs (the tutorials
use an AWS p3 instance with 4 GPUs).

The tutorial code is hosted at this `github
repo <https://github.com/suraj813/distributed-pytorch>`__. Clone the repo and
follow along!

Tutorial sections
-----------------

0. Introduction This page
1. `What is DDP? <ddp_theory.html>`__ Gently introduces what DDP is doing
   under the hood
2. `Single-node Multi-GPU training <2_multigpu.html>`__ Training models
   using multiple GPUs on a single machine
3. `Fault-tolerant distributed training <3_fault_tolerance.html>`__
   Making your distributed training job robust with torchrun
4. `Multi-node training <4_multinode.html>`__ Training models using
   multiple GPUs on multiple machines
5. `Training a GPT model with DDP <5_minGPT.html>`__ “Real-world”
   example of training a `minGPT <https://github.com/karpathy/minGPT>`__
   model with DDP
