# Viam Triton MLModel Service

A Viam provided MLModel service resource module backed by NVIDIA's [Triton Inference
Server](https://developer.nvidia.com/triton-inference-server).
Configure this MLModel service as a modular resource on your robot with a [Jetson board](https://docs.viam.com/build/configure/components/board/jetson/) to deploy MLModels to your robot faster while consuming less computer power.

## Requirements

A NVIDIA Jetson Orin board with the following installed:

1. [Jetpack 5](https://developer.nvidia.com/embedded/jetpack)
2. NVIDIA Container Runtime

Run the following to install `nvidia-jetpack` and `nvidia-container` on your robot's computer:

```sh { class="command-line" data-prompt="$"}
sudo apt-get install nvidia-jetpack nvidia-container
```

Pull the triton module docker container:

```sh { class="command-line" data-prompt="$"}
docker pull ghcr.io/viamrobotics/viam-mlmodelservice-triton:latest
```

Examine the output to find the exact tag associated with latest.
Use this as `"version"` in [configuration](https://docs.viam.com/registry/examples/triton/#configuration).

Then, if you haven't done so already, create a new robot in [the Viam app](https://app.viam.com).
[Install `viam-server` on the board](https://docs.viam.com/get-started/installation/prepare/jetson-agx-orin-setup/) and connect to the robot.

## Build and Run

To use this module, follow these instructions to [add a module from the Viam Registry](https://docs.viam.com/registry/configure/#add-a-modular-resource-from-the-viam-registry) and select the `mlmodelservice:triton` model from the `mlmodelservice-triton-jetpack` module.

## Configure your Triton MLModel Service

> [!Note]
> Before configuring your MLModel service module, you must [create a robot](https://docs.viam.com/manage/fleet/robots/#add-a-new-robot).

Navigate to the **Config** tab of your robot’s page in [the Viam app](https://app.viam.com/).
Click on the **Services** subtab and click **Create service**.
Select **ML Model**, then search for the `mlmodelservice:triton` model. Give your resource a name of your choice and click **Create**.

Switch to **Raw JSON** mode and add the following to your `"modules"` array:

```json
{
  "type": "registry",
  "name": "viam_mlmodelservice-triton-jetpack",
  "module_id": "viam:mlmodelservice-triton-jetpack",
  "version": "0.4.0"
}
```

Replace the value of the "version" field with the value you determined above with the docker pull command.

Add the following to your `"services"` array:

```json
{
 "name": "my-triton-module",
  "type": "mlmodel",
  "namespace": "rdk",
  "model": "viam:mlmodelservice:triton"
}
```

The model will now be configured with a card like the following:

![The triton service card in the Viam app config builder, showing deployment options.](https://docs.viam.com/registry/triton/triton-config-builder.png)

Note that the parameters shown, `"model_path"`, `"label_path"`, and `"num_threads"` are not applicable for this module, and should be left blank.
Ignore this card's interface and move to [creating a model repository](#create-a-repository-to-store-the-ml-model-to-deploy).

> [!NOTE]  
> For more information, see [Configure a Robot](https://docs.viam.com/manage/configuration/).


### Create a repository to store the ML model to deploy

Currently, you must manually create a Triton [model repository](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/user_guide/model_repository.html).
On your robot's Jetson computer, create a [structured repository](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/user_guide/model_repository.html) under the `~/.viam` directory.
The exact subpath under `~/.viam` does not matter.

For example, after unpacking the module, to add the [EfficientDet-Lite4 Object Detection](https://tfhub.dev/tensorflow/efficientdet/lite4/detection/2) model, place the model repository under `~/.viam/triton/repository`:

```sh { class="command-line" data-prompt="$"}
$ tree ~/.viam
~/.viam
├── cached_cloud_config_05536cf6-f8a6-464f-b05c-bf1e57e9d1d9.json
└── triton
    └── repository
        └── efficientdet-lite4-detection
            ├── 1
            │   └── model.savedmodel
            │       ├── saved_model.pb
            │       └── variables
            │           ├── variables.data-00000-of-00001
            │           └── variables.index
            └── config.pbext
```

The `config.pbext` file must exist, but at least for TensorFlow models it can be empty.
For other types of models, please consult the [Triton Server model repository documentation](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/user_guide/model_repository.html) for details on the correct contents of the `config.pbext` file.
The version here is `1` but it can be any positive integer.
Newer versions will be preferred by default.

### Attributes

After creating your model repository, configure the required attributes to deploy your model on your robot.
The following attributes are available for the MLModel service `viam:mlmodelservice:triton`:

| Name | Type | Inclusion | Description |
| ---- | ---- | --------- | ----------- |
| `model_name` | string | **Required** | The model to be loaded from the model repository. |
| `model_repository_path` | string | **Required** | The container-side path to a model repository. Note that this must be a subdirectory of the `$HOME/.viam` directory of the user running `viam-server`. |
| `backend_directory` | string | Optional | A container side path to the TritonServer "backend" directory. You normally do not need to override this; the build will set it to the backend directory of the Triton Server installation in the container. You may set it if you wish to use a different set of backends. |
| `model_version` | int | Optional | The version of the model to be loaded. If not specified, the module will use the newest version of the model named by model_name.<br><br>Default: `-1` (newest) |
| `preferred_input_memory_type` | string | Optional | One of `cpu`, `cpu-pinned`, or `gpu`. This controlls the type of memory that will be allocated by the module for input tensors. If not specified, this will default to `cpu` if no CUDA-capable devices are detected at runtime, or to `gpu` if CUDA-capable devices are found.|
| `preferred_input_memory_type_id` | int | Optional | CUDA identifier on which to allocate gpu or cpu-pinned input tensors. You probably don't need to change this unless you have multiple GPUs<br><br>Default: `0` (first device) |
| `tensor_name_remappings` | obj | Optional | Provides two dictionaries under the `input` and `output` keys that rename the models' tensors. Other Viam services, like the [vision service](/ml/vision/) may expect to work with tensors with particular names. Use this map to rename the tensors from the loaded model as needed to meet those requirements.<br><br>Default: `{}` |

### Example configurations

An example minimal configuration would look like this, within your robot’s "services" array:

```json {class="line-numbers linkable-line-numbers"}
{
  "type": "mlmodel",
  "attributes": {
    "model_name": "efficientdet-lite4-detection",
    "model_repository_path": "/path/to/.viam/triton/repository"
  },
  "model": "viam:mlmodelservice:triton",
  "name": "mlmodel-effdet-triton"
}
```

An example detailed configuration with optional parameters specified would look like this:



```json {class="line-numbers linkable-line-numbers"}
{
  "type": "mlmodel",
  "attributes": {
    "backend_directory": "/opt/tritonserver/backends",
    "model_name": "efficientdet-lite4-detection",
    "model_version": 1,
    "model_repository_path": "/path/to/.viam/triton/repository",
    "preferred_input_memory_type_id": 0,
    "preferred_input_memory_type": "gpu",
    "tensor_name_remappings": {
      "outputs": {
        "output_3": "n_detections",
        "output_0": "location",
        "output_1": "score",
        "output_2": "category"
      },
      "inputs": {
        "images": "image"
      }
    }
  },
  "model": "viam:mlmodelservice:triton",
  "name": "mlmodel-effdet-triton"
}
```

## Next Steps

- Use the [`jtop`](https://github.com/rbonghi/jetson_stats) utility on the Jetson line to monitor GPU usage to validate that Triton is accelerating inference through the GPU.

- Use the `viam:mlmodelservice:triton` modular service to perform inference with the machine learning models available in the Triton service's model repository on your robot.

- Create a [vision service](https://docs.viam.com/ml/vision/) with a configuration in your `"services"` array like the following:

  ```json {class="line-numbers linkable-line-numbers"}
  {
    "attributes": {
      "mlmodel_name": "mlmodel-effdet-triton"
    },
    "model": "mlmodel",
    "name": "vision-effdet-triton",
    "type": "vision"
  }
  ```

    You can now connect this vision service to a [transform camera](https://docs.viam.com/build/configure/components/camera/transform/), or get detections programmatically through one of Viam's [client SDKs](https://docs.viam.com/build/program/).
