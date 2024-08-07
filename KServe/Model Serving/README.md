# Model Serving
> 다양한 Model을 Serving하고 이를 REST API Call 하는 작업들을 담았다.

### Index :
1. [__Tensorflow Serving__](#tfserving)
2. [__TorchServe__](#torchserve)
3. [__Scikit Learn Server__](#sklearn_server)

# Tensorflow Serving (TFServing) <a name="tfserving" />
> Tensorflow 기반의 Model Serving 과정에서의 기록

Tensorflow Model을 Serving하면서 부딪혔던 문제들에 대해서 간단하게 기록으로 남기고자 한다. (Model, Logic 같은 부분은 모름) [Tensorflow 공식 홈페이지]() 에 나온 Tutorial을 기반으로 Serving을 한 것이므로, 복잡한 설정은 하지 않았으며, 어떤 부분들을 신경써야하는지도 가이드를 해본다.

Tensorflow Model Save 과정에서 _Keras_ 를 이용하는 방법이 있고, _Saved Model_ 을 이용하는 방법이 있다. 우선 테스트를 하면서 본 것은 Keras - Save는 동작이 되질 않았다. (Runtime Version의 문제일 수도 있고, Directory 구성이 잘못된 것일 수도 있음. 추가적인 Test가 필요함)

우선 Tensorflow의 경우에는 Directory를 Mount하는데, Directory는 다음과 같은 구조를 가져야 한다.

```
${MODEL_NAME}
--- ${NUMBER}
     --- assets
     --- ${MODEL}.pb
     --- variables
         --- variables.index
         ...
```

가장 헷갈리기 쉬운 것은 ```${NUMBER}``` 부분인데, 그냥 모델을 만들고 나서 아무렇게나 저장하면 Model을 정상적으로 Serving하지 않는다. 모델이 떨어진 directory 안에는 반드시 __Number__ 가 지정된 디렉토리가 있어야하며, 해당 하위 디렉토리에 모델이 저장되어있어야 한다.

Model이 규격에 맞추어 정상적으로 만들어졌다면, Model을 Serving하는데, PVC에 저장되어있는 Model을 불러온다고 가정한 Inference YAML file이다.

Test 했던 Model도 정리해두었다.

- Test 목록 :

1. [MobileNet](https://github.com/JeonDongcheol/StudyMyWork/tree/main/KServe/Model%20Serving/MobileNet)
2. [DNN 기반 Image Classification](https://github.com/JeonDongcheol/StudyMyWork/tree/main/KServe/Model%20Serving/DNN_Image_Classification)

Model Serving은 다음과 같이 정의했다.

```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  annotations:
    # sidecar injection false를 설정하지 않으면 RBAC(Role Bind Access Control) Error가 발생한다.
    sidecar.istio.io/inject: false
  name: ${INFERENCE_NAME}
  namespace: ${NAMESPACE}
spec:
  predictor:
    tensorflow:
      storageUri: "pvc://${PVC_NAME}/${MODEL_PATH}"
    resources:
      limits:
        cpu: 500m
        memory: 500Mi
      requests:
        cpu:300m
        memory: 350Mi
    # Runtime Version : 대부분은 Docker Hub에 있는 Image Tag를 말한다.
    # Tensorflow Serving의 경우에는 docker.io/tensorflow/serving 의 Image를 가져온다.
    runtimeVersion: 2.9.0
```

추가적인 문제 사항으로는 ```signature_name``` 이 설정되어 있지 않으면 default로 찾는데, 그것도 지정되어있지 않는 Model도 있었다. 설정되어 있지 않으면 signature name은 반드시 확인해보도록 한다.

------------------------

# Pytorch Serving (TorchServe) <a name="torchserve" />
> TorchServe 기반의 Model Serving 과정에서의 기록

Pytorch는 생각보다 번거로운 작업들이 필요했다. Model을 ```pt``` file 혹은 ```pth``` file로 만들었다고 해도, 그것을 그대로 올리면 되는 것이 아니었다. (Tensorflow는 규격만 잘 맞추면 잘 올라갔다...) Pytorch를 TorchServe 기반으로 Serving하는 과정은 다음과 같다.

1. Model Save (pt, pth...)
2. Torch Model Archive
3. config.properties
4. config / model-store directory

이 과저에서 가장 중요한건 Model을 만들고 Torch Model Archive를 통해 Model, Handler, Model File 등을 ```~.mar``` 로 압축을 해야한다는 점이었다. ```mar``` file만 있다고 model을 Serving할 수 있는 것도 아니었다. ```config.properties``` 를 통해 Model에 대한 설정을 해준다.

위의 작업이 끝났으면 Directory를 분할해주는데, ```config``` directory 안에는 ```config.properties``` 를 담아주고, ```model-store``` 에는 config.properties 에 설정해 두었던 ```Model MAR``` file을 담는다.

만약 ```Handler``` , ```Model``` Code file을 정상적으로 작업했다면, 정상적으로 Serving이 될 것이다.

TorchServe Github에 나와있는 __MNIST__ 를 기반으로 우선 Test를 해보았는데, GCS(Google Cloud Storage)에 있는 모델을 가져오는 것은 정상적으로 수행했으나, 해당 Model을 PVC(Persistent Volume Claim)에 올려서 Serving했을 때는 정상적으로 올라가지 않았다. 그 문제도 해결해보았다.

- Test 목록 :

1. [Pytorch MNIST Model Serving](https://github.com/JeonDongcheol/StudyMyWork/tree/main/KServe/Model%20Serving/MNIST)

TorchServe Github에 나와있는 Handler 및 Model file python code로 Serving을 하게 되면 __500 Internal Server__ Error가 발생했다. 구조를 뜯어보니 Handler와 Model Code 일부가 누락되거나 잘못된 부분이 있어서 그런 것이었다. 그래서 내 기준 정상적으로 동작했을 때의 코드를 올렸다.

- Handler : ```mnist_handler.py```
- Model File : ```mnist.py```
- Model pt File : ```mnist.pt```

3개의 File을 기반으로 MAR 파일을 생성해준다. PVC 사용하는 사람을 위한 Auto Archiver도 있었지만, 일단 Jupyter Notebook에서 ```pip3 install torch-model-archiver``` 를 수행하고 Model을 압축했다.

```shell
# MNIST Model Archive
torch-model-archiver --model-name ${MODEL_NAME} --version 1.0 --model-file ${PATH}/mnist.py --serialized-file ${PATH}/mnist_cnn.pt --handler ${PATH}/mnist_handler.py
```

위에서 언급했던대로, Serving Model의 Architecture는 다음과 같다.

```
${DIRECTORY_NAME}
   --- config
       --- config.properties
   --- model-store
       --- ${MODEL_NAME}.mar
```

해당 Model을 Serving하는 YAML file 구조는 다음과 같다.

```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  annotations:
    sidecar.istio.io/inject: 'false'
  name: ${INFERENCE_NAME}
  namespace: ${NAMESPACE}
spec:
  predictor:
    pytorch:
      # Resource는 알아서 할당
      resources:
        limits:
          cpu: 500m
          memory: 1Gi
        requests:
          cpu: 250m
          memory: 500Mi
      storageUri: pvc://${PVC_NAME}/${PATH}
```

Model이 정상적으로 올라가면, Prediction을 수행하는데, 주의할 점은 TorchServe의 경우에는 ```model-store``` 안에 여러 개의 Model이 들어갈 수 있다. 그래서 Prediction REST API가 반드시 __${MODEL_NAME}__ 으로 지정되어야 한다. ${INFERENCE_NAME} 으로 endpoint를 주게 되면 이상한 결과를 보게 될 것이다.

#### Reference:

- [Tensorflow Serving : DNN Image Classification](https://www.tensorflow.org/guide/saved_model)
- [Tensorflow Serving : MobileNet](https://www.tensorflow.org/tutorials/keras/classification)
- [TorchServe Github - MNIST](https://github.com/pytorch/serve/tree/master/examples/image_classifier/mnist)
