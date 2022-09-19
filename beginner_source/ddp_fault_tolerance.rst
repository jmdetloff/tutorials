`Introduction <ddp_series_intro.html>`__ \|\| `What is DDP <ddp_theory.html>`__ \|\| `Single-node
Multi-GPU training <ddp_multigpu.html>`__ \|\| **Fault
Tolerance** \|\| `Multi-node
training <intermediate/ddp_multinode.html>`__ \|\| `mingpt training <intermediate/ddp_minGPT.html>`__


Fault-tolerant Distributed Training with ``torchrun``
=====================================================

Authors: `Suraj Subramanian <https://github.com/suraj813>`__

In distributed training, a single process failure can
disrupt the entire training job. Since the susceptibility for failure can be higher here, making your training
script robust is particularly important here. You might also prefer your training job to be *elastic* i.e. the ability to increase or decrease the number
of processes in realtime while running the training job.

PyTorch has a utility ``torchrun`` that provides fault-tolerance as well
as elastic training. When a failure occurs, torchrun logs the errors and
attempts to automatically restart all the processes from the last saved
“snapshot” of the training job. 

The snapshot saves more than just the model state; it can include
details about the number of epochs run, optimizer states or any other
mutable attribute of the training job necessary for its continuity.


What you will learn
-------------------
-  Launching multi-GPU training jobs with ``torchrun``
-  Saving and loading snapshots of your training job
-  Structuring your training script for graceful restarts


View the code used in this video: https://github.com/suraj813/distributed-pytorch/blob/main/multigpu_torchrun.py


.. raw:: html

   <embed video>


Why use ``torchrun``
~~~~~~~~~~~~~~~~~~~~

``torchrun`` handles the minutiae of distributed training so that you
don't need to. For instance,
- You don't need to set environment
variables or explicitly pass the ``rank`` and ``world_size``; torchrun
assigns this along with several other `environment
variables <https://pytorch.org/docs/stable/elastic/run.html#environment-variables>`__.
- No need to call ``mp.spawn`` in your script; you only need a generic
``main()`` entrypoint, and launch the script with ``torchrun``. This way
the same script can be run in non-distributed as well as single-node and
multinode setups. 
- Gracefully restarting training from the last saved training
snapshot


Graceful restarts
~~~~~~~~~~~~~~~~~~~~~
For graceful restarts, it helps to have the following structure:

.. code:: python

   def main():
     load_snapshot(snapshot_path)
     initialize()
     train()

   def train():
     for batch in iter(dataset):
       train_step(batch)

       if should_checkpoint:
         save_snapshot(snapshot_path)

If a failure occurs, torchrun will terminate all the processes and restart them. 
Each process entrypoint first loads and initializes the last saved snapshot, and continues training from there.
So at any failure, you only lose the training progress from the last saved snapshot. 

In elastic training, whenever there are any membership changes (adding or removing nodes), torchrun will terminate and spawn processes
on available devices. Having this structure ensures your training job can continue without manual intervention.





Diff for `multigpu.py <https://github.com/suraj813/distributed-pytorch/blob/main/multigpu.py>`__ v/s `multigpu_torchrun.py <https://github.com/suraj813/distributed-pytorch/blob/main/multigpu_torchrun.py>`__
-----------------------------------------------------------

Process group initialization
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: diff

   - def ddp_setup(rank, world_size):
   + def ddp_setup():
   -     """
   -     Args:
   -         rank: Unique identifier of each process
   -         world_size: Total number of processes
   -     """
   -     os.environ["MASTER_ADDR"] = "localhost"
   -     os.environ["MASTER_PORT"] = "12355"
   -     init_process_group(backend="nccl", rank=rank, world_size=world_size)
   +     init_process_group(backend="nccl")

-  ``torchrun`` assigns ``RANK`` and ``WORLD_SIZE`` automatically,
   amongst `other env
   variables <https://pytorch.org/docs/stable/elastic/run.html#environment-variables>`__

Use Torchrun-provided env variables
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: diff

   - self.gpu_id = gpu_id
   + self.gpu_id = int(os.environ["LOCAL_RANK"])

Saving and loading snapshots
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: diff

   + def _save_snapshot(self, epoch):
   +     snapshot = {}
   +     snapshot["MODEL_STATE"] = self.model.module.state_dict()
   +     snapshot["EPOCHS_RUN"] = epoch
   +     torch.save(snapshot, "snapshot.pt")
   +     print(f"Epoch {epoch} | Training snapshot saved at snapshot.pt")

   + def _load_snapshot(self, snapshot_path):
   +     snapshot = torch.load(snapshot_path)
   +     self.model.load_state_dict(snapshot["MODEL_STATE"])
   +     self.epochs_run = snapshot["EPOCHS_RUN"]
   +     print(f"Resuming training from snapshot at Epoch {self.epochs_run}")

Regularly storing all the relevant information in snapshots allows our
training job to seamlessly resume after an interruption.

Loading a snapshot in the Trainer constructor
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: diff

   class Trainer:
      def __init__(self, snapshot_path, ...):
      ...
   +  if os.path.exists(snapshot_path):
   +     self._load_snapshot(snapshot_path)
      ...

When restarting an interrupted training job, your script will first try
to load a snapshot to resume training from.

Resuming training
~~~~~~~~~~~~~~~~~

.. code:: diff

   def train(self, max_epochs: int):
   -  for epoch in range(max_epochs):
   +  for epoch in range(self.epochs_run, max_epochs):
         self._run_epoch(epoch)

Training can resume from the last epoch run, instead of starting all
over from scratch.

Running the script
~~~~~~~~~~~~~~~~~~

.. code:: diff

   if __name__ == "__main__":
      import sys
      total_epochs = int(sys.argv[1])
      save_every = int(sys.argv[2])
   -  world_size = torch.cuda.device_count()
   -  mp.spawn(main, args=(world_size, total_epochs, save_every,), nprocs=world_size)
   +  main(save_every, total_epochs)

Call your entrypoint function as usual; ``torchrun`` automatically
spawns the processes.

.. code:: diff

   - python multigpu.py 50 10
   + torchrun --standalone --nproc_per_node=4 multigpu_torchrun.py 50 10

Further Reading
---------------

-  `Multi-node training with DDP <intermediate/ddp_multinode.html>`__  (next tutorial in this series)
-  `Multi-GPU training with DDP <ddp_multigpu.html>`__ (previous tutorial in this series)
-  `torchrun <https://pytorch.org/docs/stable/elastic/run.html>`__
-  `Torchrun launch
   options <https://github.com/pytorch/pytorch/blob/bbe803cb35948df77b46a2d38372910c96693dcd/torch/distributed/run.py#L401>`__
-  `Migrating from torch.distributed.launch to
   torchrun <https://pytorch.org/docs/stable/elastic/train_script.html#elastic-train-script>`__
