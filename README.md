# Mxnet Image Classification Example
This is Mxnet image classification training container with recording time of the metrics.

It uses only simple multilayer perceptron network (mlp).

If you want to read more about this example, visit official [incubator-mxnet](https://github.com/apache/incubator-mxnet/tree/v0.9.3/example/image-classification) github repository.


## Creating a Persistent Volume Claim

Content of **mxnet-mnist-pvc.yaml**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mxnet-mnist-pvc
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
```
Create PVC:
```bash
kubectl create -n dev1 -f https://raw.githubusercontent.com/SANDataHPE/Kubeflow-Pipelines/main/katib-job-mxnet-mnist-pvc/mxnet-mnist-pvc.yaml
```

## Utility Pod to copy dataset to Persistent Volume

Content of **dataaccess.yaml**
```yaml
apiVersion: v1
kind: Pod
metadata:
    name: dataaccess
spec:
    containers:
    - name: alpine
      image: alpine:latest
      command: ['sleep', 'infinity']
      volumeMounts:
      - name: mypvc
        mountPath: /data
    volumes:
    - name: mypvc
      persistentVolumeClaim:
        claimName: mxnet-mnist-pvc
```
Create utility pod:
```bash
kubectl create -n dev1 -f https://raw.githubusercontent.com/SANDataHPE/Kubeflow-Pipelines/main/katib-job-mxnet-mnist-pvc/dataaccess.yaml
```
## Prepare dataset
* **_Non Air Gap Enviornment/Proxy Enviornment_**
  ```bash
  kubectl -n dev1 exec -it dataaccess sh
  ```
  Execute the below commands to download the dataset.
  ```bash
  #Creating MXNET-MNIST directory in PV
  mkdir -p /data/MXNET-MNIST
  
  #Export proxy if required 
  export http_proxy=http://x.x.x.x 
  export https_proxy=http://x.x.x.x
  
  #Downloading datasetes
  wget http://yann.lecun.com/exdb/mnist/train-labels-idx1-ubyte.gz -P /data/MXNET-MNIST
  wget http://yann.lecun.com/exdb/mnist/train-images-idx3-ubyte.gz -P /data/MXNET-MNIST
  wget http://yann.lecun.com/exdb/mnist/t10k-labels-idx1-ubyte.gz -P /data/MXNET-MNIST
  wget http://yann.lecun.com/exdb/mnist/t10k-images-idx3-ubyte.gz -P /data/MXNET-MNIST
  ```
* **_Air Gap Enviornment_**

  Download below files locally and using winscp copy to kubernetes master host. </br>
  [train-labels-idx1-ubyte.gz](http://yann.lecun.com/exdb/mnist/train-labels-idx1-ubyte.gz) </br>
  [train-images-idx3-ubyte.gz](http://yann.lecun.com/exdb/mnist/train-images-idx3-ubyte.gz) </br>
  [t10k-labels-idx1-ubyte.gz](http://yann.lecun.com/exdb/mnist/t10k-labels-idx1-ubyte.gz) </br>
  [t10k-images-idx3-ubyte.gz](http://yann.lecun.com/exdb/mnist/t10k-images-idx3-ubyte.gz)
  <br>
  ```bash
  #Copy the files to PV using utility pod from kubernetes master host.
  kubectl -n dev1 cp MXNET-MNIST/t10k-images-idx3-ubyte.gz dataaccess:/data/MXNET-MNIST/t10k-images-idx3-ubyte.gz
  kubectl -n dev1 cp MXNET-MNIST/train-images-idx3-ubyte.gz dataaccess:/data/MXNET-MNIST/train-images-idx3-ubyte.gz
  kubectl -n dev1 cp MXNET-MNIST/t10k-labels-idx1-ubyte.gz dataaccess:/data/MXNET-MNIST/t10k-labels-idx1-ubyte.gz
  kubectl -n dev1 cp MXNET-MNIST/train-labels-idx1-ubyte.gz dataaccess:/data/MXNET-MNIST/train-labels-idx1-ubyte.gz
  ```
  Verify data is copied or not.
  ```bash
  kubectl -n dev1 exec -t dataaccess -c alpine  -- ls -lrt /data/MXNET-MNIST
  ```

## Create Experiment

Content of **mxnet-mnist-pvc.yaml**
```yaml
apiVersion: "kubeflow.org/v1beta1"
kind: Experiment
metadata:
  namespace: dev1
  name: mxnet-mnist-pvc
spec:
  objective:
    type: maximize
    goal: 0.99
    objectiveMetricName: Validation-accuracy
    additionalMetricNames:
      - Train-accuracy
  algorithm:
    algorithmName: random
  parallelTrialCount: 3
  maxTrialCount: 3
  maxFailedTrialCount: 3
  parameters:
    - name: --lr
      parameterType: double
      feasibleSpace:
        min: "0.01"
        max: "0.03"
    - name: --num-layers
      parameterType: int
      feasibleSpace:
        min: "2"
        max: "5"
    - name: --optimizer
      parameterType: categorical
      feasibleSpace:
        list:
          - sgd
          - adam
          - ftrl
  trialTemplate:
    goTemplate:
        rawTemplate: |-
          apiVersion: batch/v1
          kind: Job
          metadata:
            name: {{.Trial}}
            namespace: {{.NameSpace}}
          spec:
            template:
              spec:
                volumes:
                  - name: dataset
                    persistentVolumeClaim:
                      claimName: mxnet-mnist-pvc
                containers:
                  - name: training-container
                    image: ilovepython/katib-mxnet-mnist-pvc:v0.1
                    command:
                      - 'python3'
                      - '/opt/mxnet-mnist/mxnet-mnist.py'
                      - '--batch-size=64'
                      - '--data-dir=/dataset/MXNET-MNIST'
                      {{- with .HyperParameters}}
                      {{- range .}}
                      - "{{.Name}}={{.Value}}"
                      {{- end}}
                      {{- end}}
                    volumeMounts:
                      - mountPath: "/dataset"
                        name: dataset
                restartPolicy: Never

```
Copy the content of **mxnet-mnist-pvc.yaml** and submit the experiment from Katib HyperParameter Tunning tab in Kubeflow UI. 

## Experiment Screenshots


