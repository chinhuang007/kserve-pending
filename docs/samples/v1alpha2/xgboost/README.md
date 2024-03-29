## Creating your own model and testing the XGBoost server.

To test the XGBoost Server, first we need to generate a simple XGBoost model using Python. 

```python
import xgboost as xgb
from sklearn.datasets import load_iris
import os

model_dir = "."
BST_FILE = "model.bst"

iris = load_iris()
y = iris['target']
X = iris['data']
dtrain = xgb.DMatrix(X, label=y)
param = {'max_depth': 6,
            'eta': 0.1,
            'silent': 1,
            'nthread': 4,
            'num_class': 10,
            'objective': 'multi:softmax'
            }
xgb_model = xgb.train(params=param, dtrain=dtrain)
model_file = os.path.join((model_dir), BST_FILE)
xgb_model.save_model(model_file)
```

Then, we can install and run the [XGBoost Server](../../../python/xgbserver) using the generated model and test for prediction. Models can be on local filesystem, S3 compatible object storage, Azure Blob Storage, or Google Cloud Storage.

```shell
python -m xgbserver --model_dir /path/to/model_dir --model_name xgb
```

We can also use the inbuilt sklearn support for sample datasets and do some simple predictions

```python
from sklearn.datasets import load_iris
import requests

iris = load_iris()
X = iris['data']

request = [X[0].tolist()]
formData = {
    'instances': request
}
res = requests.post('http://localhost:8080/v1/models/xgb:predict', json=formData)
print(res)
print(res.text)
```

## Predict on a InferenceService using XGBoost Server

## Setup
1. Your ~/.kube/config should point to a cluster with [KFServing installed](https://github.com/kubeflow/kfserving/#install-kfserving).
2. Your cluster's Istio Ingress gateway must be [network accessible](https://istio.io/latest/docs/tasks/traffic-management/ingress/ingress-control/).

## Create the InferenceService

Apply the CRD
```
kubectl apply -f xgboost.yaml
```

Expected Output
```
$ inferenceservice.serving.kserve.io/xgboost-iris created
```

## Run a prediction
The first step is to [determine the ingress IP and ports](../../../../README.md#determine-the-ingress-ip-and-ports) and set `INGRESS_HOST` and `INGRESS_PORT`

```
MODEL_NAME=xgboost-iris
INPUT_PATH=@./iris-input.json
SERVICE_HOSTNAME=$(kubectl get inferenceservice xgboost-iris -o jsonpath='{.status.url}' | cut -d "/" -f 3)
curl -v -H "Host: ${SERVICE_HOSTNAME}" http://${INGRESS_HOST}:${INGRESS_PORT}/v1/models/$MODEL_NAME:predict -d $INPUT_PATH
```

Expected Output

```
*   Trying 169.63.251.68...
* TCP_NODELAY set
* Connected to 169.63.251.68 (169.63.251.68) port 80 (#0)
> POST /models/xgboost-iris:predict HTTP/1.1
> Host: xgboost-iris.default.svc.cluster.local
> User-Agent: curl/7.60.0
> Accept: */*
> Content-Length: 76
> Content-Type: application/x-www-form-urlencoded
>
* upload completely sent off: 76 out of 76 bytes
< HTTP/1.1 200 OK
< content-length: 27
< content-type: application/json; charset=UTF-8
< date: Tue, 21 May 2019 22:40:09 GMT
< server: istio-envoy
< x-envoy-upstream-service-time: 13032
<
* Connection #0 to host 169.63.251.68 left intact
{"predictions": [1.0, 1.0]}
```

## Run XGBoost InferenceService with your own image
Since the KFServing XGBoost image is built from a specific version of `xgboost` pip package, sometimes it might not be compatible with the pickled model
you saved from your training environment, however you can build your own XGBServer image following [this instruction](../../../python/xgbserver/README.md#building-your-own-xgboost-server-docker-image).

To use your XGBServer image:
- Add the image to the KFServing [configmap](../../../config/configmap/inferenceservice.yaml)
```yaml
        "xgboost": {
            "image": "<your-dockerhub-id>/kfserving/xbgserver",
        },
```
- Specify the `runtimeVersion` on `InferenceService` spec
```yaml
apiVersion: "serving.kserve.io/v1alpha2"
kind: "InferenceService"
metadata:
  name: "xgboost-iris"
spec:
  default:
    predictor:
      sklearn:
        storageUri: "gs://kfserving-samples/models/xgboost/iris"
        runtimeVersion: X.X.X
```