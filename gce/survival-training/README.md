# Compute Engine survival training

Training jobs that survive Compute Engine shutdowns

- - -

[Google Compute Engine](https://cloud.google.com/compute/docs/) (Compute Engine) gives you direct access to the computational backbone of Google Cloud Platform. You can use Compute Engine to create virtual machines provisioned with the resources of your choosing, to [satisfy the constraints of any quotas that you define](https://cloud.google.com/compute/quotas). This is especially useful for training machine learning models, because you can access resources beyond those available to you physically. 

Compute Engine offers preemptible GPU instances. If you design your trainer appropriately, your cost per training step could be a fraction of what it would be on a dedicated instance. Whether you use preemptible GPU instances or dedicated instances, you must consider how to design your trainer to ensure that your jobs survive routine maintenance shutdowns.

Dedicated GPU-enabled Compute Engine instances go down regularly for maintenance, and they restart without reinitializing the processes that were running on them before the shutdown. Similarly, preemptible VMs can be shut down at any time with very little advance warning, interrupting whichever processes were running on them. They also have a maximum uptime of 24 hours.

This guide provides you with a template to design a trainer that is robust against the behaviours of both dedicated GPU instances and preemptible instances using exactly the same semantics - the main issue you have to tackle is of your job surviving through a shutdown.


## Survival strategy

We will adopt a relatively simple survival strategy. Throughout a training run, we will expect trainers to persist the state of the model being trained in checkpoints. We will survive our training job using only Compute Engine primitives to start them back up from the most recently saved checkpoint after a VM restart.

Note that, under this policy, we are willing to lose some amount of training computation -- specifically, the computation done between when the most recent checkpoint was stored and the time of a potential failure.

You can use Compute Engine semantics to go one step further and ensure that a checkpoint is stored on maintenance or preemptions. However, this implementation is slightly trickier and it puts more burden on your checkpointing setup. Given a suitable frequency of checkpointing, the solution presented here should cover most needs.

There are five components to our resilient training jobs:

1. [Compute Engine custom images](https://cloud.google.com/compute/docs/images#custom_images)

1. A trainer command line interface

1. A delivery mechanism to inject your trainer into a Compute Engine instance

1. [Compute Engine startup scripts](https://cloud.google.com/compute/docs/startupscript)

1. [Compute Engine instance metadata](https://cloud.google.com/compute/docs/storing-retrieving-metadata)


### Prerequisites

As a part of this process, if you have not already done so, you will be prompted to enable the Compute Engine API.

We will be working in large part through the [Google Cloud SDK](https://cloud.google.com/sdk/). If you have not already done so, it is worth installing.


### Custom images

Your training job may require specific versions of libraries, like [tensorflow](https://www.tensorflow.org/) or [scikit-learn](http://scikit-learn.org). Additionally, in order to take advantage of instances with attached GPUs, you will have to have hardware-specific drivers installed on the instance. [Custom images](https://cloud.google.com/compute/docs/images#custom_images) are a convenient solution to this problem of environment reproduction within a Compute Engine virtual machine.

If your training environments have additional requirements beyond the OS, it is highly recommended that you create a [custom image](https://cloud.google.com/compute/docs/images#custom_images), which is a convenient template for virtual machine configuration that you can use to create new instances without having to configure each one individually. The examples following this strategy section will demonstrate how to create custom images.


### Trainer CLI

Our strategy will have the Compute Engine instance automatically kick off your training job on startup. As part of this kick off, it will have to provide the job with the appropriate parameters -- how many steps to train for, how often to checkpoint the model, which data to train the model with, what hyperparameters should be used in training the model, etc.

If you intend to do this more than once, it is helpful to assert a common semantical system across all your trainers. We provide a [boilerplate CLI](./wrapper/train.py) in this repo. This interface is very similar to that provided by [TensorFlow estimators](https://www.tensorflow.org/programmers_guide/estimators), but can be used even with other frameworks. In the sections that follow, we will provide you with both TensorFlow and with scikit-learn examples that demonstrate its use.


### Delivery mechanism options

We need a mechanism by which we can get our trainer code into the Compute Engine instance. There are several solutions available to us:

1. Store the trainer application on the custom image from which we create the Compute Engine instance.

1. Store the code in a [Cloud Storage](https://cloud.google.com/storage/) bucket and have the startup script download it to the instance before doing anything else.

1. Store the code in a [Cloud Source Repository](https://cloud.google.com/source-repositories/) and have the startup script clone it to the instance before doing anything else.

1. Clone an existing repository from [GitHub](https://github.com/). (Caution: if working with private repos, this would require you to set up a git credentials helper in your VM image).

In our sample deployments (below), we will demonstrate some of these options. Changing between them will generally involve only minor tweaks to the startup scripts and instance metadata, so it should not be too challenging to generalize to any of the others. If you do have problems, please raise an issue.


### Startup script

We use [startup scripts](https://cloud.google.com/compute/docs/startupscript) to begin a new training job or to resume an existing training job. Compute Engine instances have a convenient built-in mechanism that triggers a script on instance startup.

If you are unfamiliar with startup scripts, it is enough to understand what they are, and generally how they fit into the training process.


### Instance metadata

Compute Engine instances have a [metadata server that we use to store specific parameter settings](https://cloud.google.com/compute/docs/storing-retrieving-metadata#custom), or parameterizations, for training jobs. This gives you the flexibility to deploy training jobs with different parameterizations to multiple Compute Engine instances at different times.

We will use instance metadata to store information such as:

+ model hyperparameters

+ locations of training data

+ locations of evaluation data

We will store these types of training information as instance metadata before we start our training jobs. Once the Compute Engine instance starts up, our startup scripts make use of this metadata immediately.

Adapting [our example startup script](./Compute Engine/startup.sh) to your training requires minimal changes. Instance metadata and startup scripts work together to start the training process in your Compute Engine instance.


## Examples

The following examples demonstrate the use of the framework provided here:

+ [TensorFlow estimator](./README-tf-estimator.md)

+ [sklearn model](./README-sklearn.md) - work in progress

- - -

##### Notes and references

+ The trainer command-line interface suggested here is modeled on the [the Cloud ML Engine trainer interface](https://cloud.google.com/ml-engine/docs/packaging-trainer). In case you want to perform distributed training with TensorFlow models, using this command-line interface for your trainer makes it easier to move your training from Compute Engine to Cloud ML Engine.
