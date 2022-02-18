# MNIST demo for SKAI

## Preparation

Install skai dashboard can refer this [doc](https://github.com/skai-x/dashboard/tree/master/charts/skai-dashboard).

## Develop

Please refer to the documentation [here](https://github.com/skai-x/elastic-jupyter-operator#deploy) to deploy elastic-jupyter-operator on your Kubernetes cluster.

Then you can create a simple Jupyter notebook with all components in one pod, like this:

```bash
$ cat >> notebook.yaml << EOF  
apiVersion: kubeflow.tkestack.io/v1alpha1
kind: JupyterNotebook
metadata:
  name: jupyternotebook-simple
spec:
  auth:
    mode: disable
  template:
    metadata:
      labels:
        notebook: simple
    spec:
      containers:
        - name: notebook
          image: ghcr.io/skai-x/ml-demo-jupyter:latest
          command: ["tini", "-g", "--", "start-notebook.sh"]
EOF

$ kubectl apply -f ./notebook.yaml
$ kubectl port-forward deploy/jupyternotebook-simple 8888:8888
```

Then you can open the URL `http://127.0.0.1:8888/` to get the simple Jupyter notebook instance.

## Train

In this section, we create a simple pipeline for training mnist model.

### Create CodeSet

We will create a codeset cr, this cr describe the code will be used in training pod.

```bash
$ cat >> tf-mnist.yaml << EOF 
apiVersion: train.skai.io/v1alpha1
kind: CodeSet
metadata:
  name: mnist-tensorflow 
spec:
  description: "machine learning demo in skai github repo"
  git:
    branch: "main"
    url: https://github.com/skai-x/ml-demo.git 
EOF

$ kubectl apply -f tf-mnist.yaml

$ kubectl get codeset
NAME               CREATED AT
mnist-tensorflow   2021-12-07T12:58:18Z
```

### Create Dataset

We will create a dataset cr, this cr describe the the configuration of dataset that will be used in training pod.

```bash
cat >> cos_mnist_dataset.yaml << EOF
apiVersion: train.skai.io/v1alpha1
kind: Dataset 
metadata:
  name: mnist-cos 
spec:
  description: train mnist dataset in cos
  type: Object
  objectStorage:
    type: "COS"
    url: "http://cos.ap-shanghai.myqcloud.com"
    bucket: "rdmatest-1251707795"
    path: "/"
    accessID: "xx"
    secretKey: "xx"
    options: "-oallow_other"
EOF

$ kubectl apply -f cos_mnist_dataset.yaml 

$ kubectl get dataset
NAME        CREATED AT
mnist-cos   2022-01-04T07:47:21Z
```

### Create tensorflow train job

we will create tensorflow train job for train mnist demo, will use code and dataset in above definition.

```bash
$ cat >> tf-job.yaml << EOF
apiVersion: train.skai.io/v1alpha1
kind: TrainJob
metadata:
  name: mnist-cos-train-tf
spec:
  description: "The tensorflow trainjob created by xieydd , using dataset mnist-cos and codeset mnist"
  jobType: "tfjob"
  tensorBoard:
    eventLogDir: /tmp/xieyuandong/mnist-cos-train-tf/ 
  instanceInfos:
  - role: "Worker"
    replicas: 1 
    image: tensorflow/tensorflow:1.15.5-py3-jupyter 
    resource:
      cpuRequest: 2
      cpuLimit: 2 
      memoryRequest: 2Gi
      memoryLimit: 2Gi
  enableDistributedTraining: false 
  datasetInfo:
    name: "mnist-cos"
    mountPath: "/cos"
  codesetInfo:
    name: "mnist-tensorflow"
    mountPath: "/root"
  workingDir: "/root"
  command: "python /root/ml-demo/mnist/tensorflow/mnist_with_summaries/mnist_with_summaries_savedmodel.py --log_dir=/tmp/xieyuandong/mnist-cos-train-tf/logs --learning_rate=0.01 --batch_size=150 --data_dir=/cos/MNIST/raw/ --saved_model_dir=/cos/mnist_seldon"
EOF

$ kubectl apply -f tf-job.yaml

$ kubectl get trainjob
NAME                 CREATED AT
mnist-cos-train-tf   2022-01-04T08:27:29Z
```

**NOTICE** If github repo can't be used, you can change codeset git url to `https://gitee.com/xieyuandong/tf-mnist.git` and change train job command to `python /root/tf-mnist/mnist_with_summaries_savedmodel.py --log_dir=/tmp/xieyuandong/mnist-cos-train-tf/logs --learning_rate=0.01 --batch_size=150 --data_dir=/cos/MNIST/raw/ --saved_model_dir=/cos/mnist_seldon`.

## Inference

### Install Seldon with Istio

```shell
$ kubectl create namespace istio-system
```

Using TKE Mesh create service mesh. notice set name `seldon-gateway` in `istio-system`, create ingress gateway watch default namespace application. Then create Gateway `seldon-gateway` in default namespace and set port is 8080

```shell
$ git clone https://github.com/SeldonIO/seldon-core.git /tmp/seldon-core
$ kubectl create namespace seldon-system
$ helm install seldon-core /tmp/seldon-core/helm-charts/seldon-core-operator \
    --set usageMetrics.enabled=true \
    --namespace seldon-system \
    --set istio.enabled=true
```

### Mnist Demo
1. Create mnist SeldonDeployment
```shell
$ kubectl apply -f mnist-seldon-tfserving.yaml 
```

2. Inference

```shell
$ kubectl port-forward $(kubectl get pods -l istio=ingressgateway -n istio-system -o jsonpath='{.items[0].metadata.name}') -n istio-system 8004:8080
export SELDON_URL=localhost:8004
$ python mnist-inference.py 
```
You can find the result of the inference and the image of your input data.
