.. _train-pytorch-transformers:

Get Started with Hugging Face Transformers
==========================================

This tutorial walks through the process of converting an existing Hugging Face Transformers script to use Ray Train.

Learn how to:

1. Configure a :ref:`training function <train-overview-training-function>` to report metrics and save checkpoints.
2. Configure :ref:`scaling <train-overview-scaling-config>` and CPU or GPU resource requirements for your training job.
3. Launch your distributed training job with a :class:`~ray.train.torch.TorchTrainer`.

Quickstart
----------

For reference, the final code follows:

.. testcode::
    :skipif: True

    from ray.train.torch import TorchTrainer
    from ray.train import ScalingConfig

    def train_func(config):
        # Your Transformers training code here.
    
    scaling_config = ScalingConfig(num_workers=2, use_gpu=True)
    trainer = TorchTrainer(train_func, scaling_config=scaling_config)
    result = trainer.fit()

1. `train_func` is the Python code that executes on each distributed training worker.
2. :class:`~ray.train.ScalingConfig` defines the number of distributed training workers and whether to use GPUs.
3. :class:`~ray.train.torch.TorchTrainer` launches the distributed training job.

Compare a Hugging Face Transformers training script with and without Ray Train.

.. tabs::

    .. group-tab:: Hugging Face Transformers

        .. This snippet isn't tested because it doesn't use any Ray code.

        .. testcode::
            :skipif: True

            # Adapted from Hugging Face tutorial: https://huggingface.co/docs/transformers/training

            import numpy as np
            import evaluate
            from datasets import load_dataset
            from transformers import (
                Trainer,
                TrainingArguments,
                AutoTokenizer, 
                AutoModelForSequenceClassification,
            )

            # Datasets
            dataset = load_dataset("yelp_review_full")
            tokenizer = AutoTokenizer.from_pretrained("bert-base-cased")

            def tokenize_function(examples):
                return tokenizer(examples["text"], padding="max_length", truncation=True)

            small_train_dataset = dataset["train"].select(range(1000)).map(tokenize_function, batched=True)
            small_eval_dataset = dataset["test"].select(range(1000)).map(tokenize_function, batched=True)

            # Model
            model = AutoModelForSequenceClassification.from_pretrained(
                "bert-base-cased", num_labels=5
            )

            # Metrics
            metric = evaluate.load("accuracy")

            def compute_metrics(eval_pred):
                logits, labels = eval_pred
                predictions = np.argmax(logits, axis=-1)
                return metric.compute(predictions=predictions, references=labels)

            # Hugging Face Trainer
            training_args = TrainingArguments(
                output_dir="test_trainer", evaluation_strategy="epoch", report_to="none"
            )

            trainer = Trainer(
                model=model,
                args=training_args,
                train_dataset=small_train_dataset,
                eval_dataset=small_eval_dataset,
                compute_metrics=compute_metrics,
            )

            # Start Training
            trainer.train()

                

    .. group-tab:: Hugging Face Transformers + Ray Train

        .. code-block:: python
            :emphasize-lines: 11-13, 15-18, 55-72

            import numpy as np
            import evaluate
            from datasets import load_dataset
            from transformers import (
                Trainer,
                TrainingArguments,
                AutoTokenizer, 
                AutoModelForSequenceClassification,
            )

            import ray.train.huggingface.transformers
            from ray.train import ScalingConfig
            from ray.train.torch import TorchTrainer

            # [1] Encapsulate data preprocessing, training, and evaluation 
            # logic in a training function
            # ============================================================
            def train_func(config):
                # Datasets
                dataset = load_dataset("yelp_review_full")
                tokenizer = AutoTokenizer.from_pretrained("bert-base-cased")

                def tokenize_function(examples):
                    return tokenizer(examples["text"], padding="max_length", truncation=True)

                small_train_dataset = dataset["train"].select(range(1000)).map(tokenize_function, batched=True)
                small_eval_dataset = dataset["test"].select(range(1000)).map(tokenize_function, batched=True)

                # Model
                model = AutoModelForSequenceClassification.from_pretrained(
                    "bert-base-cased", num_labels=5
                )

                # Evaluation Metrics
                metric = evaluate.load("accuracy")

                def compute_metrics(eval_pred):
                    logits, labels = eval_pred
                    predictions = np.argmax(logits, axis=-1)
                    return metric.compute(predictions=predictions, references=labels)

                # Hugging Face Trainer
                training_args = TrainingArguments(
                    output_dir="test_trainer", evaluation_strategy="epoch", report_to="none"
                )

                trainer = Trainer(
                    model=model,
                    args=training_args,
                    train_dataset=small_train_dataset,
                    eval_dataset=small_eval_dataset,
                    compute_metrics=compute_metrics,
                )

                # [2] Report Metrics and Checkpoints to Ray Train
                # ===============================================
                callback = ray.train.huggingface.transformers.RayTrainReportCallback()
                trainer.add_callback(callback)

                # [3] Prepare Transformers Trainer
                # ================================
                trainer = ray.train.huggingface.transformers.prepare_trainer(trainer)

                # Start Training
                trainer.train()

            # [4] Define a Ray TorchTrainer to launch `train_func` on all workers
            # ===================================================================
            ray_trainer = TorchTrainer(
                train_func, scaling_config=ScalingConfig(num_workers=4, use_gpu=True)
            )
            ray_trainer.fit()


Set up a training function
--------------------------

First, update your training code to support distributed training. 
You can begin by wrapping your code in a :ref:`training function <train-overview-training-function>`:

.. testcode::
    :skipif: True

    def train_func(config):
        # Your Transformers training code here.

This function executes on each distributed training worker. Ray Train sets up the distributed 
process group on each worker before entering this function.

Put all the logic into this function, including dataset construction and preprocessing, 
model initialization, transformers trainer definition and more.

.. note::

    If you are using Hugging Face Datasets or Evaluate, make sure to call ``datasets.load_dataset`` and ``evaluate.load`` 
    inside the training function. Don't pass the loaded datasets and metrics from outside of the training 
    function, because it might cause serialization errors while transferring the objects to the workers.


Report checkpoints and metrics
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

To persist your checkpoints and monitor training progress, add a 
:class:`ray.train.huggingface.transformers.RayTrainReportCallback` utility callback to your Trainer. 


.. code-block:: diff

     import transformers
     from ray.train.huggingface.transformers import RayTrainReportCallback

     def train_func(config):
         ...
         trainer = transformers.Trainer(...)
    +    trainer.add_callback(RayTrainReportCallback())
         ...


Reporting metrics and checkpoints to Ray Train ensures that you can use Ray Tune and :ref:`fault-tolerant training <train-fault-tolerance>`. 
Note that the :class:`ray.train.huggingface.transformers.RayTrainReportCallback` only provides a simple implementation, and you can :ref:`further customize <train-dl-saving-checkpoints>` it.


Prepare a Transformers Trainer
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Finally, pass your Transformers Trainer into
:meth:`~ray.train.huggingface.transformers.prepare_trainer` to validate 
your configurations and enable Ray Data Integration. 


.. code-block:: diff

     import transformers
     import ray.train.huggingface.transformers

     def train_func(config):
         ...
         trainer = transformers.Trainer(...)
    +    trainer = ray.train.huggingface.transformers.prepare_trainer(trainer)
         trainer.train()
         ...


Configure scale and GPUs
------------------------

Outside of your training function, create a :class:`~ray.train.ScalingConfig` object to configure:

1. `num_workers` - The number of distributed training worker processes.
2. `use_gpu` - Whether each worker should use a GPU (or CPU).

.. testcode::

    from ray.train import ScalingConfig
    scaling_config = ScalingConfig(num_workers=2, use_gpu=True)


For more details, see :ref:`train_scaling_config`.

Launch a training job
---------------------

Tying this all together, you can now launch a distributed training job 
with a :class:`~ray.train.torch.TorchTrainer`.

.. testcode::
    :hide:

    from ray.train import ScalingConfig

    train_func = lambda: None
    scaling_config = ScalingConfig(num_workers=1)

.. testcode::

    from ray.train.torch import TorchTrainer

    trainer = TorchTrainer(train_func, scaling_config=scaling_config)
    result = trainer.fit()

Refer to :ref:`train-run-config` for more configuration options for `TorchTrainer`.

Access training results
-----------------------

After training completes, a :class:`~ray.train.Result` object is returned which contains
information about the training run, including the metrics and checkpoints reported during training.

.. testcode::

    result.metrics     # The metrics reported during training.
    result.checkpoint  # The latest checkpoint reported during training.
    result.path     # The path where logs are stored.
    result.error       # The exception that was raised, if training failed.

.. TODO: Add results guide

Next steps
---------- 

After you have converted your Hugging Face Transformers training script to use Ray Train:

* See :ref:`User Guides <train-user-guides>` to learn more about how to perform specific tasks.
* Browse the :ref:`Examples <train-examples>` for end-to-end examples of how to use Ray Train.
* Dive into the :ref:`API Reference <train-api>` for more details on the classes and methods used in this tutorial.
