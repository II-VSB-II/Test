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
kubectl create -n <namespace> -f http://
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
```
kubectl create -n <namespace> -f http://
```
## Prepare dataset
* Non Air Gap Enviornment/Proxy Enviornment
```
kubectl -n <namespace> exec -it dataaccess sh
```
Execute the below commands to download the dataset.
```
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
* Air Gap Enviornment
