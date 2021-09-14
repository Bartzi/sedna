# Collaboratively Train Yolo-v5 Using MistNet on COCO128 Dataset

This case introduces how to train a federated learning job with an aggregation algorithm named MistNet in MNIST
handwritten digit classification scenario. Data is scattered in different places (such as edge nodes, cameras, and
others) and cannot be aggregated at the server due to data privacy and bandwidth. As a result, we cannot use all the
data for training. In some cases, edge nodes have limited computing resources and even have no training capability. The
edge cannot gain the updated weights from the training process. Therefore, traditional algorithms (e.g., federated
average), which usually aggregate the updated weights trained by different edge clients, cannot work in this scenario.
MistNet is proposed to address this issue.

MistNet partitions a DNN model into two parts, a lightweight feature extractor at the edge side to generate meaningful
features from the raw data, and a classifier including the most model layers at the cloud to be iteratively trained for
specific tasks. MistNet achieves acceptable model utility while greatly reducing privacy leakage from the released
intermediate features.

## Object Detection Experiment

> Assume that there are two edge nodes and a cloud node. Data on the edge nodes cannot be migrated to the cloud due to privacy issues.
> Base on this scenario, we will demonstrate the mnist example.

### Prepare Nodes

```
CLOUD_NODE="cloud-node-name"
EDGE1_NODE="edge1-node-name"
EDGE2_NODE="edge2-node-name"
```

### Install Sedna

Follow the [Sedna installation document](/docs/setup/install.md) to install Sedna.

### Prepare Dataset

Download [dataset](https://github.com/ultralytics/yolov5/releases/download/v1.0/coco128.zip) 

Create data interface for ```EDGE1_NODE```.

```shell
mkdir -p /data/1
cd /data/1
wget https://github.com/ultralytics/yolov5/releases/download/v1.0/coco128.zip
unzip coco128.zip -d COCO
```

Create data interface for ```EDGE2_NODE```.

```shell
mkdir -p /data/2
cd /data/2
wget https://github.com/ultralytics/yolov5/releases/download/v1.0/coco128.zip
unzip coco128.zip -d COCO
```

### Prepare Images

This example uses these images:

1. aggregation worker: ```kubeedge/sedna-example-federated-learning-mistnet-yolo-aggregato:v0.4.0```
2. train worker: ```kubeedge/sedna-example-federated-learning-mistnet-yolo-client:v0.4.0```

These images are generated by the script [build_images.sh](/examples/build_image.sh).

### Create Federated Learning Job

#### Create Dataset

create dataset for `$EDGE1_NODE` and `$EDGE2_NODE`

```bash
kubectl create -f - <<EOF
apiVersion: sedna.io/v1alpha1
kind: Dataset
metadata:
  name: "coco-dataset-1"
spec:
  url: "/data/1/COCO"
  format: "dir"
  nodeName: $EDGE1_NODE
EOF
```

```bash
kubectl create -f - <<EOF
apiVersion: sedna.io/v1alpha1
kind: Dataset
metadata:
  name: "coco-dataset-2"
spec:
  url: "/data/2/COCO"
  format: "dir"
  nodeName: $EDGE2_NODE
EOF
```

#### Create Model
create the directory `/model` and `/pretrained` in `$EDGE1_NODE` and `$EDGE2_NODE`.
```bash
mkdir -p /model
mkdir -p /pretrained
```

create the directory `/model` and `/pretrained` in the host of `$CLOUD_NODE` (download links [here](https://kubeedge.obs.cn-north-1.myhuaweicloud.com/examples/yolov5_coco128_mistnet/yolov5.pth))


```bash
# on the cloud side
mkdir -p /model
mkdir -p /pretrained
cd /pretrained
wget https://kubeedge.obs.cn-north-1.myhuaweicloud.com/examples/yolov5_coco128_mistnet/yolov5.pth
```

create model

```bash
kubectl create -f - <<EOF
apiVersion: sedna.io/v1alpha1
kind: Model
metadata:
  name: "yolo-v5-model"
spec:
  url: "/model/yolov5.pth"
  format: "pth"
EOF

kubectl create -f - <<EOF
apiVersion: sedna.io/v1alpha1
kind: Model
metadata:
  name: "yolo-v5-pretrained-model"
spec:
  url: "/pretrained/yolov5.pth"
  format: "pth"
EOF
```

### Create a secret with your S3 user credential. (Optional)

```shell
kubectl create -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
  annotations:
    s3-endpoint: s3.amazonaws.com 
    s3-usehttps: "1" 
stringData: 
  ACCESS_KEY_ID: XXXX
  SECRET_ACCESS_KEY: XXXXXXXX
EOF
```

#### Start Federated Learning Job

```bash
kubectl create -f - <<EOF
apiVersion: sedna.io/v1alpha1
kind: FederatedLearningJob
metadata:
  name: yolo-v5
spec:
  pretrainedModel: # option
    name: "yolo-v5-pretrained-model"
  transmitter: # option
    ws: { } # option, by default
    s3: # optional, but at least one
      aggDataPath: "s3://sedna/fl/aggregation_data"
      credentialName: mysecret
  aggregationWorker:
    model:
      name: "yolo-v5-model"
    template:
      spec:
        nodeName: $CLOUD_NODE
        containers:
          - image: kubeedge/sedna-example-federated-learning-mistnet-yolo-aggregator:v0.4.0
            name: agg-worker
            imagePullPolicy: IfNotPresent
            env: # user defined environments
              - name: "cut_layer"
                value: "4"
              - name: "epsilon"
                value: "100"
              - name: "aggregation_algorithm"
                value: "mistnet"
              - name: "batch_size"
                value: "32"
            resources: # user defined resources
              limits:
                memory: 8Gi
  trainingWorkers:
    - dataset:
        name: "coco-dataset-1"
      template:
        spec:
          nodeName: $EDGE1_NODE
          containers:
            - image: kubeedge/sedna-example-federated-learning-mistnet-yolo-client:v0.4.0
              name: train-worker
              imagePullPolicy: IfNotPresent
              args: [ "-i", "1" ]
              env: # user defined environments
                - name: "cut_layer"
                  value: "4"
                - name: "epsilon"
                  value: "100"
                - name: "aggregation_algorithm"
                  value: "mistnet"
                - name: "batch_size"
                  value: "32"
                - name: "learning_rate"
                  value: "0.001"
                - name: "epochs"
                  value: "1"
              resources: # user defined resources
                limits:
                  memory: 2Gi
    - dataset:
        name: "coco-dataset-2"
      template:
        spec:
          nodeName: $EDGE2_NODE
          containers:
            - image: kubeedge/sedna-example-federated-learning-mistnet-yolo-client:v0.4.0
              name: train-worker
              imagePullPolicy: IfNotPresent
              args: [ "-i", "2" ]
              env: # user defined environments
                - name: "cut_layer"
                  value: "4"
                - name: "epsilon"
                  value: "100"
                - name: "aggregation_algorithm"
                  value: "mistnet"
                - name: "batch_size"
                  value: "32"
                - name: "learning_rate"
                  value: "0.001"
                - name: "epochs"
                  value: "1"
              resources: # user defined resources
                limits:
                  memory: 2Gi
EOF
```

