# Train and Serve a Model

## Prerequisite

Please make sure both [SKAI/dashboard](https://github.com/skai-x/dashboard) and [SKAI/model-pool](https://github.com/skai-x/model-pool) are installed.

Please also follow this [tutorial](https://github.com/skai-x/ml-demo/blob/main/mnist/tensorflow/dist-mnist/README.md) to create CodeSet and DataSet.

## Create ModelVersion

Use the [modelversion.yaml](https://github.com/skai-x/ml-demo/blob/main/mnist/tensorflow/with-model-pool/modelversion.yaml) to create a ModelVersion.

A PersistentVolumeClaim named `mlp-v1-pvc` will be created. User can probe the such information from the status of the modelversion.

## Train

Submit the [train.yaml](https://github.com/skai-x/ml-demo/blob/main/mnist/tensorflow/with-model-pool/train.yaml) to launch the training job.

## Serving

Submit the [serving.yaml](https://github.com/skai-x/ml-demo/blob/main/mnist/tensorflow/with-model-pool/serving.yaml) to launch the serving job.
