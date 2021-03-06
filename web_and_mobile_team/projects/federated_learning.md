## Introduction

_Last modified: January 30th, 2020_

To my knowledge, this will be the first open-source system for private federated learning on server, web, and mobile anywhere on the planet. Our target platforms will include worker libraries written for servers (Python), web browsers (Javascript), Android devices (Kotlin), and iOS devices (Swift). We will do this by allowing for PySyft "plans" to be executed by a remote worker other than PySyft itself.

For the web, we must run TensorFlow.js as our deep learning framework. However, in order to run PyTorch plans in TensorFlow.js, we must first serialize the operations as a list, and then convert each operation into the appropriate commands available in TensorFlow.js. This translation layer will be described in more detail below.

Conversely, for Android and iOS, we have chosen to utilize PyTorch Mobile. PyTorch Mobile allows us to run plans for both training and inference of PyTorch models via a wrapper library (written in Java) for Android and using LibTorch (PyTorch’s internal C++ library) for iOS. There is not currently a finished Objective-C or Swift wrapper that has been released by the PyTorch team. It’s worth noting that for the mobile workers, plans will need to be converted into TorchScript in order to be executed - the Javascript worker does not have this concern.

In addition to "plans" that represent a batch of operations to be executed on a single worker, PySyft introduces the concept of a "protocol" that represents a distributed computation graph with defined worker roles. A "protocol" deployed to workers should allow executing SMPC algorithms such as those necessary for secure aggregation. Secure aggregation will require workers, or a subset of workers, to communicate directly - this can be accomplished by using WebRTC to establish a peer-to-peer data channel between workers. This allows for an added layer of protection to the workers that is separate from differential privacy which is applied as part of the eventual model update process on PyGrid.

**\*Patrick Cason**<br />
Web and Mobile Team Lead\*

## Terminology

Let’s start off by defining a few terms that will be regularly used throughout this document:

- **Model** - a standard machine learning model
- **Operation/Command** - a single function in PyTorch, TensorFlow, or TensorFlow.js
- **Plan** - a serialized list of operations that can be executed by a worker
- **Protocol** - a graph of operations on a list of Workers
- **Worker** - any server, device, or browser that is capable of running a PySyft model for training and inference
- **VirtualWorker** - a virtualized Python worker that’s capable of pretending to be a separate server-based worker
- **PyTorch** - a deep learning library written in Python
- **LibTorch** - the tensor-based mathematical library written in C++ that is called by PyTorch
- **TorchScript** - a generated language that allows for a PyTorch method to be simplified into an object representing PyTorch operations. In the context of our system, we will serialize PySyft plans into TorchScript so that they may be executed by mobile worker libraries.
- **Wrapper library** - a library written in one language that allows for execution in another language. For instance, there is a Java wrapper for LibTorch which is written in C++.
- **Training plan** - the plan that a worker will run on their server, device, or browser
- **Averaging plan** - the plan run by the central server that determines how the models should be aggregated to eventually update the global model
- **Cycle** - a single run of the federated learning process from workers pulling down the model and training it until the global model update process is finished
- **Secure aggregation** - given a state where multiple parties each have one or more numbers, these techniques allow them to average their numbers without any party seeing any other party’s raw input to the average.
- **Secure multi-party computation (SMPC)** - a class of algorithms allowing for multiple parties to collaboratively compute a function without revealing their inputs to the function to each other.

We will also use this section for defining each of the components of the OpenMined federated learning system:

- **PySyft** - a framework for privacy-preserving machine learning that hooks both PyTorch and TensorFlow
- **PyGrid** - a library consisting of multiple services that allows for PySyft workers to be deployed across a network. It’s also responsible to manage the FL process and host models, plans, configurations, and tensors.
- **syft-proto** - a Protocol Buffer interface used between PySyft and the various worker libraries that allows for class definitions to be shared across platforms
- **syft.js** - the official Javascript worker library for PySyft, running in the web browser
- **KotlinSyft** - the official Kotlin worker library for PySyft, running on Android devices
- **SwiftSyft** - the official Swift worker library for PySyft, running on iOS devices
- **Threepio** - a small command translation library that allows for PyTorch or TensorFlow commands to be translated to and from TensorFlow.js

## Overview

[View a diagram of this process](https://drive.google.com/file/d/1xWKA9YEJCdUq-y49Sx1vSlR5n-48v7dH/view?usp=sharing)

The OpenMined federated learning workflow is loosely inspired Google’s federated learning workflow, but we provide a bit more generalization since we don’t also control the operating system itself. Our workflow consists of the following steps:

### 1. Design

A developer designs their FL model using PyTorch in PySyft. This process involves the creation of a model, training plan, averaging plan, and (optionally) a protocol.

The training plan will be the function that is inevitably run by a worker, taking the model and a batch of training data as input. The plan’s logic includes forward and backward propagation, as well as the weight update step.

The averaging plan will instruct PyGrid on how to average models that are returned from the workers. This is also known as the "model update" plan whereby PyGrid receives trained models from workers, averages them, and then updates the global model before beginning another cycle.

If a protocol is written, this will be executed by the worker libraries after they’ve finished executing the training plan. This will allow the workers to split off into groups to perform some sort of computation amongst themselves using WebRTC as a peer-to-peer data transfer layer. This allows for secure aggregation to be performed and later decrypted by PyGrid.

Below is a proposed sample code block of how this might work:

```py
import syft as sy
import torch

hook = sy.TorchHook(torch)
hook.local_worker.is_client_worker = False

class Net(sy.Plan):
    def __init__(self):
        super(Net, self).__init__()
        self.linear1 = torch.nn.Linear(10, 5)
        self.linear2 = torch.nn.Linear(5, 1)

    def forward(self, x):
        x = self.linear1(x)
        x = torch.nn.functional.relu(x)
        x = self.linear2(x)
        return x

# Model
model = Net()

x = torch.zeros(1, 10)
y = torch.zeros(1, 1)

# Training plan
@func2plan(model=model, data=x, target=y, optimizer=nn.SGD(model.parameters(), lr=0.01))
def training_plan(model, data, target, optimizer):
    data, target = data, target
    optimizer.zero_grad()
    output = model(data)
    loss = F.nll_loss(output, target)
    loss.backward()
    optimizer.step()

# Averaging plan
@func2plan
def averaging_plan(model_params):
    sum = model_params[0]
    for params in model_params[1:]:
        sum += params
    avg = sum / len(model_params)
    return avg

# Secure aggregation protocol
worker1 = sy.VirtualWorker(hook=hook, id="worker1")
worker2 = sy.VirtualWorker(hook=hook, id="worker2")
worker3 = sy.VirtualWorker(hook=hook, id="worker3")

worker1_model = worker1.Promise.Model()
worker2_model = worker2.Promise.Model()
worker3_model = worker3.Promise.Model()

worker1_shares = worker1_model.share(worker1, worker2, worker3)
worker2_shares = worker2_model.share(worker1, worker2, worker3)
worker3_shares = worker3_model.share(worker1, worker2, worker3)

new_model_shares = (worker1_shares + worker2_shares + worker3_shares) / 3

new_model_shares["worker1"].declare_output()
new_model_shares["worker2"].declare_output()
new_model_shares["worker3"].declare_output()
secure_aggregation_protocol = sy.Protocol(worker1, worker2, worker3)
```

### 2. Test

At this point, a developer will test their code against VirtualWorkers in PySyft, allowing them to locally simulate the process of deploying their model to the edge. The developer will also want to define their initial model parameters (e.g. model name, version, hyperparameters, etc.) and federated learning configuration (e.g. the number of cycles, maximum number of workers, etc.).

This process will create a network of VirtualWorkers in PySyft, send the training plan to each of them, execute the model on each of the VirtualWorkers, push the results back to PySyft, run the averaging plan to update the global model, and finally return the model back to the developer for inspection.

```py
# These attributes should correspond to inputs in the training_plan
model_config = {
  name: "my-federated-model",
  version: "0.1.0",
  batch_size: 32,
  lr: 0.01,
  optimizer: "SGD"
}

fl_config = {
  max_workers: 100,
  pool_selection: "random",  # or "iterate"
  num_cycles: 5,
  do_not_reuse_workers_until_cycle: 4,
  cycle_length: 8 * 60 * 60  # 8 hours
}

sy.test_federated_training(
  model=model,
  training_plan=training_plan,
  averaging_plan=averaging_plan,
  protocol=secure_aggregation_protocol,
  model_config=model_config,
  fl_config=fl_config
)
```

### 3. Host

Now that the model has been developed locally, it’s time to host the model on PyGrid. This will allow for training cycles to begin on end-user devices. The developer needs to connect to an existing PyGrid gateway and send the model, training plan, averaging plan, optional protocol, and various configurations to be stored in PyGrid properly.

```py
import grid as gr

pygrid = gr.WebsocketGridClient(
  hook,
  id="my-name",
  address="https://localhost:3000"
)

pygrid.connect()

pygrid.host_federated_training(
  model=model,
  training_plan=training_plan,
  averaging_plan=averaging_plan,
  protocol=secure_aggregation_protocol,
  model_config=model_config,
  fl_config=fl_config
)
```

### 4. Execute

Each worker is a library that may be integrated within the context of a web or mobile app. This integration will be relatively straight-forward, requiring no more than a few lines of code. The following is an example of how the syft.js Javascript worker might integrate into an application:

```js
import syft from 'syft.js';
import tf from '@tensorflow/tfjs';

const worker = new syft({
  url: 'https://localhost:3000'
});

// You may set up multiple jobs.
const job = worker.newJob({
  model: 'my-federated-model',
  version: '0.1.0'
});

// 1. Determine if there is a cycle that the worker can join. If there is a cycle available, it will automatically download the model, plan, and config from PyGrid. If there isn't a cycle available, this code block will automatically retry at the requested timestamp.

// NOTE FOR MOBILE WORKERS: This is the place to check for a wifi connection, whether or not the device is charging, and whether or not the user is likely asleep.
job.start();

// 2. Once the worker has been approved to participate in a cycle and the model, plan, and config have been downloaded...
job.on('ready', ({ model, model_config, protocol }) => {
  const { batch_size, optimizer } = model_config;

  // Load your data and targets
  const rawData = [image1, image2, image3, image4, image5, ...];
  const rawTargets = ['dog', 'cat', 'cat', 'neither', 'dog', ...];

  // Batch your data and targets
  const data = [];
  const targets = [];

  while(rawData.length) {
    data.push(rawData.splice(0, batch_size));
  }

  while(rawTargets.length) {
    targets.push(rawTargets.splice(0, batch_size));
  }

  // 3. Execute the plan in batches.
  for(let i = 0; i < rawData.length; i++) {
    job.executeTrainingPlan(model, data[i], target[i], optimizer);
  }

// 4. Once the training plan has been executed, execute the protocol if there is one. This will spawn WebRTC and send this job's model to other workers to be securely aggregated. Afterwards, or assuming there isn't a protocol, report the result back to PyGrid.
  if(protocol) {
    job.executeProtocol().then(() => {
      job.report();
    });
  }
  else {
    job.report();
  }
});
```

One initial detail worth noting is that it’s assumed a worker could be running multiple jobs at the same time. A syft worker will need to intelligently observe each job as unique, handling the current training state and other various resources and tasks independent of other jobs. It’s worth noting that mobile phones will not likely have the compute resources necessary for running multiple jobs simultaneously. Because of this, we should allow for multiple concurrent jobs to be executed, but warn the developer at compile time that this is very bad practice for mobile.

Upon first glance, the example above is performing a lot of "magic" behind the scenes. Once calling the `start()` method, the worker will notify PyGrid that it is available for training and send with it some other information related to the worker: ping, average download speed, average upload speed, and the format that the worker likes to receive a plan (list of operations or TorchScript). It’s worth noting that we want to send as little information to PyGrid as possible, only what is strictly required to ensure a reasonably paced training cycle. **Workers will not send information related to wifi connectivity, charging state, or asleep/awakeness of the user. It will be the responsibility of the client-side mobile developer to determine what criteria is appropriate for when model training should take place.**

It’s entirely possible that after calling `start()` that PyGrid requests for the worker to keep waiting and check back in at a later time. PyGrid will need to serve a timestamp to the worker notifying it of when to check back in. It should be noted that the worker will not need to call `start()` again at this later point; instead, the worker will simply go into "sleep mode" until that time. This will disable the socket connection and start a timer.

When the `on('ready')` event listener is triggered, this means that the worker has been chosen to participate by PyGrid and the model, training plan, and model configuration have been downloaded. Again, this magically happens in the background. At this point, the developer will likely batch their input and label data sets, and the worker may begin to run `executeTrainingPlan()`.

Once the plan is fully trained, we must check if there is a protocol the worker needs to run. Assuming there isn’t, we can run `report()` which will then upload the resulting model back to PyGrid for aggregation into the global model. However, if a protocol is present, it will need to be run by calling `executeProtocol()`. The trained model will automatically be passed into the `executeProtocol()` function in the background.

You can expect the code sample above to change slightly between the various worker libraries. It’s worth considering that the mobile workers are also plagued by different concerns like detecting when the user is awake or asleep, determining charge status, and being limited to execution time limits for background tasks to name a few. It will be up to the mobile libraries themselves to do the detection of this criteria (battery level, charging/not-charging, internet connection speed, and asleep/awake), and we will provide reasonable default levels. However, any of these criteria should be allowed to be overridden by the developer as per their specific use-case.

### 5. Aggregate

At this point, a model has either been trained and reported back to PyGrid or shares of a securely aggregated (and trained) model have been reported back to PyGrid. If PyGrid receives multiple shares of a model (as it would in the presence of a protocol), then it will need to combine the shares and then decrypt the result. If the report is sent within the time limit of the cycle (set in the federated learning configuration above), it will be accepted. Otherwise, it will be discarded. The worker will not be notified in either situation.

At this point, PyGrid needs to update the global model with the new weights. Google does this by using an algorithm they appropriately call "Federated Averaging". While it’s perfectly appropriate to use their averaging algorithm, we opted to allow the developer to define their own in the form of a PySyft plan, aptly called the "averaging plan". This function will go through each of the reported models and average them against the global model, which then persists as the new global model for future cycles.

It’s important to note that this will actually create a new "checkpoint" of the model. If the developer originally uploaded a model to PyGrid of name "my-federated-model" with a version of "0.1.0", then it would automatically create a checkpoint of "1". For each checkpoint after that, the value will increment. It would be beneficial for PyGrid to also allow a developer to name their checkpoints something more memorable like "latest" or "stable" or "best-version-yet".

### 6. Inspect

Depending on the federated learning configuration, the developer may opt to have multiple training cycles. In this case, a new cycle will begin at the determined interval. However, assuming we’ve finished all the cycles, the model will now be ready for inspection by the developer and may be used for inference, retrained, or whatever they desire. The developer may decide to pull a specific version of a model, or may optionally decide to even pull a specific checkpoint instead. The process of federated learning is now considered complete.

```py
import grid as gr

pygrid = gr.WebsocketGridClient(
  hook,
  id="my-name",
  address="https://localhost:3000"
)

pygrid.connect()

trained_fl_model = pygrid.get_model(name="my-federated-model", version="0.1.0", checkpoint="latest")
```

## Projects

There are many projects that we consider vital to achieving our MVP. [You may also view our current progress on Github](https://github.com/orgs/openmined/projects/13). The high-level projects we must complete are as follows:

### PySyft

- [Refactor Plans](https://github.com/OpenMined/PySyft/issues/2912)
- [Refactor Protocols](https://github.com/OpenMined/PySyft/issues/2903)
- [Refactor PyTorch tensor hooking](https://github.com/OpenMined/PySyft/issues/2991)
- [Allow PySyft to package plan operations either as a list of individual commands or as TorchScript](https://github.com/OpenMined/PySyft/issues/2994)
- [Add test federated training method to PySyft](https://github.com/OpenMined/PySyft/issues/2995)
- [Implement changes in PySyft client for federated learning](https://github.com/OpenMined/PySyft/issues/2996)
- _Optional_ - [Implement the command translation layer inside of PySyft which will allow for PySyft Tensorflow plans to be converted to PyTorch](https://github.com/OpenMined/PySyft/issues/2997)

### PyGrid

- [Add WebRTC socket signaling support to PyGrid](https://github.com/OpenMined/PyGrid/issues/412)
- [Add plans and protocols to PyGrid](https://github.com/OpenMined/PyGrid/issues/413)
- [Add host federated training method to PyGrid](https://github.com/OpenMined/PyGrid/issues/435)
- [Add get model method to PyGrid](https://github.com/OpenMined/PyGrid/issues/436)
- Allow PyGrid to serve plan operations as either a list of individual commands (syft.js) or as TorchScript (Android and iOS) depending on the requesting environment
- Implement federated learning cycles in PyGrid
- Add averaging plan and global model updating functionality to PyGrid
- _Optional_ - Add a few predefined averaging plans (like “Federated Average”) to PyGrid
- _Optional_ - Allow for periodic status updates of current FL cycles on PyGrid

### syft-proto

- Create Protobuf schemas from PySyft classes
- Set up a CI pipeline and version release system in syft-proto with Github Actions

### syft.js

- Migrate syft.js to use Protobuf classes
- Implement the command translation layer inside of syft.js
- Implement split and stitch algorithm for data channels in syft.js
- Add bandwidth and Internet connectivity test in syft.js
- Scaffold basic proposed worker API in syft.js
- Execute plans in syft.js
- Execute protocols in syft.js
- Allow for training state to be persisted to temporary storage in the event of a failure in syft.js

### KotlinSyft

- Create the basic structure of KotlinSyft
- Implement WebRTC flow in Android
- Implement Protobuf in Android
- Set up deployment to Maven
- Implement split and stitch algorithm for data channels in Android
- Add support for background task scheduling in Android
- Implement sleep/wake detection in Android
- Add support for charge detection and wifi detection in Android
- Add bandwidth and Internet connectivity test in Android
- Scaffold basic proposed worker API in Android
- Execute plans in Android
- Execute protocols in Android
- Allow for training state to be persisted to temporary storage in the event of a failure in Android

### SwiftSyft

- Create the basic structure of SwiftSyft
- Implement WebRTC flow in iOS
- Implement Protobuf in iOS
- Set up deployment to Cocoapods
- Implement split and stitch algorithm for data channels in iOS
- Add support for background task scheduling in iOS
- Implement sleep/wake detection in iOS
- Add support for charge detection and wifi detection in iOS
- Add bandwidth and Internet connectivity test in iOS
- Scaffold basic proposed worker API in iOS
- Execute plans in iOS
- Execute protocols in iOS
- Allow for training state to be persisted to temporary storage in the event of a failure in iOS

### Threepio

- Write a language-agnostic command translation layer that is tri-directional between PyTorch, TensorFlow, and TensorFlow.js

### General

- Work on multiple end-to-end demos utilizing all 3 Syft worker libraries
- Develop end-to-end testing suite for all full system testing
