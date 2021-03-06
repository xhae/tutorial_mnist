# MNIST Tensorflow Starter Code

For the English version description, please follow the link: [Link](README.en.md)

본 저장소는 MNIST 문제를 해결하기 위한 트레이닝과 머신러닝 모델 평가를 위한 기본적인 코드를 담고 있습니다.

본 저장소에 있는 코드는 데이터를 읽고, TensorFlow 모델을 훈련하고, 모델의 성능을 평가하는 일련의 과정에 대한 예제 코드입니다.
여러분은 데이터를 이용해서 [model architectures](#overview-of-models)에 주어진 것과 같은 여러 모델들을 훈련할 수 있습니다.
또한, 본 코드는 주어진 모델 외에도 여러분의 커스텀 모델에도 쉽게 적용될 수 있도록 디자인 되어 있습니다.

MNIST를 훈련시키고 평가하는 방법에는 두 가지가 있습니다: Google Cloud 위에서 작업하실 수도 있고, 여러분의 로컬 머신에서 작업하실 수도 있습니다.
본 README는 양쪽 방법 모두를 다루고자 합니다.

## Table of Contents
* [Running on Google's Cloud Machine Learning Platform](#running-on-googles-cloud-machine-learning-platform)
   * [Requirements](#requirements)
   * [Testing Locally](#testing-locally)
   * [Training on the Cloud](#training-on-features)
   * [Evaluation and Inference](#evaluation-and-inference)
   * [Inference Using Batch Prediction](#inference-using-batch-prediction)
   * [Accessing Files on Google Cloud](#accessing-files-on-google-cloud)
   * [Using Larger Machine Types](#using-larger-machine-types)
* [Running on Your Own Machine](#running-on-your-own-machine)
   * [Requirements](#requirements-1)
* [Overview of Models](#overview-of-models)
* [Overview of Files](#overview-of-files)
   * [Training](#training)
   * [Evaluation](#evaluation)
   * [Inference](#inference)
   * [Misc](#misc)
* [TODO for paticiapnts](#todo-for-participants)
* [Etc](#etc)
* [About This Project](#about-this-project)

## Running on Google's Cloud Machine Learning Platform

### Requirements

먼저 여러분의 계정이 Google Cloud Platform을 사용할 수 있도록 세팅되어야 합니다.
본 [링크](https://cloud.google.com/ml/docs/how-tos/getting-set-up)를 따라서 계정 생성 및 설정을 진행해주세요.

또한, 다음 명령들을 실행하여 Python 2.7 이상, 그리고 TensorFlow 1.0.1 이상 버전이 설치되어 있는지 확인하시기 바랍니다:

```sh
python --version
python -c 'import tensorflow as tf; print(tf.__version__)'
```

### Testing Locally
다음 gcloud 명령들은 모두 *저장소 루트* 경로에서 실행되어야 합니다.
현재 디렉토리의 위치는 명령어 'ls'를 통하여 확인하실 수 있습니다.

모델 개발 도중, 간단한 이슈들을 확인하기 위해 클라우드에 올릴 필요 없이 바로 로컬에서 테스트 해 보실 수 있습니다.
`gcloud beta ml local` 와 같은 명령어를 통해 사용하시면 되며, 예제는 다음과 같습니다:

```sh
gcloud ml-engine local train \
--package-path=mnist --module-name=mnist.train -- \
--train_data_pattern='gs://kmlc_test_train_bucket/mnist/train.tfrecords' \
--train_dir=/tmp/kmlc_mnist_train --model=LogisticModel --start_new_model
```

훈련 데이터를 현 디렉토리에 받는 법은 다음과 같습니다.

```sh
gsutil cp gs://kmlc_test_train_bucket/mnist/train.tfrecords .
```

파일을 다운로드 받고 나면, `train_data_pattern` 인자를 통하여 job에 해당 파일을 입력으로 사용할 수 있습니다.("gs://..." 경로를 통하여 원격에 있는 파일이 아닌, 로컬 파일을 지정할 수 있습니다.)

모델이 로컬에서 동작하는 것을 확인하셨다면, 아래 쓰인 방법을 통하여 클라우드에서도 시도해 보실 차례입니다.

### Training on the Cloud

이번 항목에서는 훈련 데이터와 테스트 파일들에 접근하기 위해 Google Cloud를 사용할 것입니다. 과거에 해당 계정이서 Google Cloud를 사용하신 적이 없다면 $300 상당의 Credit이 주어지게 됩니다. 다음 스텝들을 따라 가시면 프로젝트 초기화, 데이터 다운로드 및 Google Cloud ML상에서의 훈련 / 테스팅 등을 경험하실 수 있습니다.

#### Set up your Google Cloud project

[Link](https://mlchallenge2017.com/references-kr.html)에 쓰인 프로젝트 생성 및 개발환경 설정을 따라주시기 바랍니다.

#### Running training

다음 명령어는 해당 모델을 Google Cloud 위에서 훈련시킬 것입니다. 다음 명령어들은 Google Cloud Shell위에서 실행되어야 합니다.

The following commands will train a model on Google Cloud. Following commands are need to be executed on Google Cloud Shell.

```sh
BUCKET_NAME=gs://${USER}_kmlc_mnist_train_bucket
# (One Time) Create a storage bucket to store training logs and checkpoints.
gsutil mb -l us-east1 $BUCKET_NAME
# Submit the training job.
JOB_NAME=kmlc_mnist_train_$(date +%Y%m%d_%H%M%S); gcloud --verbosity=debug ml-engine jobs \
submit training $JOB_NAME \
--package-path=mnist --module-name=mnist.train \
--staging-bucket=$BUCKET_NAME --region=us-east1 \
--config=mnist/cloudml-gpu.yaml \
-- --train_data_pattern='gs://kmlc_test_train_bucket/mnist/train.tfrecords' \
--model=LogisticModel \
--train_dir=$BUCKET_NAME/kmlc_mnist_train_logistic_model
```

위 `gsutil` 명령어에서 'package-path' 플래그는 'train.py' 스크립트를 포함하고 있는 경로를 의미하며, 동시에 Cloud worker에 업로드 될 패키지를 의미하기도 합니다. 'module-name'은 실행되어야 할 파이선 스크립트를 지정하는 플래그입니다.(본 예제에서는 train module을 사용하고 있습니다.)

Google Cloud에서 업로드 후 job들이 실행되기까지는 잠시 시간이 걸립니다.
실행이 되고 나면 다음과 같은 메세지들을 보실 수 있습니다:

```
training step 270| Hit@1: 0.68 PERR: 0.52 Loss: 638.453
training step 271| Hit@1: 0.66 PERR: 0.49 Loss: 635.537
training step 272| Hit@1: 0.70 PERR: 0.52 Loss: 637.564
```

작업 도중 "ctrl-c" 를 눌러 콘솔에서 접속을 끊을 수 있습니다. 모델은 클라우드 상에서 독립적으로 계속 훈련이 진행되며, 해당 job에 대한 진행 상황을 확인하거나 멈추는 등의 작업은 [Google Cloud ML Jobs console](https://console.cloud.google.com/ml/jobs) 를 통해서 하실 수 있습니다.

여러 job들을 동시에 훈련시킬 수도 있으며, tensorboard를 통해서 모델들의 퍼포먼스를 시각화하여 보실 수 있습니다.

```sh
tensorboard --logdir=$BUCKET_NAME --port=8080
```

Tensorboard가 실행되고 나면 다음 명령어를 통해서 Tensorboard를 보실 수 있습니다: [http://localhost:8080](http://localhost:8080)
만약 Google Cloud Shell에서 실행하셨다면 콘솔 위쪽에 있는 Web Preview 버튼을 누르고 "Preview on port 8080" 메뉴를 통해서 Tensorboard를 보실 수 있습니다.

### Evaluation and Inference
다음은 모델을 Validation dataset위에서 평가하는 방법입니다:

```sh
JOB_TO_EVAL=kmlc_mnist_train_logistic_model
JOB_NAME=kmlc_mnist_eval_$(date +%Y%m%d_%H%M%S); gcloud --verbosity=debug ml-engine jobs \
submit training $JOB_NAME \
--package-path=mnist --module-name=mnist.eval \
--staging-bucket=$BUCKET_NAME --region=us-east1 \
--config=mnist/cloudml-gpu.yaml \
-- --eval_data_pattern='gs://kmlc_test_train_bucket/mnist/validation.tfrecords' \
--model=LogisticModel \
--train_dir=$BUCKET_NAME/${JOB_TO_EVAL} --run_once=True
```

다음은 모델을 Test set위에서 실행하는 방법입니다:

```sh
JOB_TO_EVAL=kmlc_mnist_train_logistic_model
JOB_NAME=kmlc_mnist_inference_$(date +%Y%m%d_%H%M%S); gcloud --verbosity=debug ml-engine jobs \
submit training $JOB_NAME \
--package-path=mnist --module-name=mnist.inference \
--staging-bucket=$BUCKET_NAME --region=us-east1 \
--config=mnist/cloudml-gpu.yaml \
-- --input_data_pattern='gs://kmlc_test_train_bucket/mnist/test.tfrecords' \
--train_dir=$BUCKET_NAME/${JOB_TO_EVAL} \
--output_file=$BUCKET_NAME/${JOB_TO_EVAL}/predictions.csv
```

위 gcloud 명령어에서 'training'이라는 부분이 다소 착각의 여지가 있습니다. 이름과는 다르게, 'training'이란 명령어는 클라우드가 호스팅하는 Python/Tensorflow 서비스를 제공하는 일을 합니다. Cloud Platform의 관점에서 보면 training과 inference는 아무 차이가 없는 일이기 때문입니다. Cloud ML Platform은 Tensorflow를 통한 예측을 위한 특별한 기능들을 제공하기는 하나 본 문서에서는 다루지 않겠습니다.

Job들이 시작되고 나면 Evaluation 코드에 대해서는 다음과 같은 메세지를 보시게 됩니다:

```
examples_processed: 1024 | global_step 447044 | Batch Hit@1: 0.782 | Batch PERR: 0.637 | Batch Loss: 7.821 | Examples_per_sec: 834.658
```

Inference 코드에 대해서는 다음과 같은 메세지를 보시게 됩니다:

```
num examples processed: 8192 elapsed seconds: 14.85
```

### Inference Using Batch Prediction
Inference를 좀 더 빠르게 진행하기 위하여 Cloud ML Batch Prediction을 사용하실 수 있습니다.

먼저, Training job이 모델을 내보내는 디렉토리를 찾아주세요:

```
gsutil list ${BUCKET_NAME}/kmlc_mnist_train_logistic_model/export
```

다음과 비슷한 출력을 보시게 될 겁니다:

```
${BUCKET_NAME}/kmlc_mnist_train_logistic_model/export/
${BUCKET_NAME}/kmlc_mnist_train_logistic_model/export/step_1/
${BUCKET_NAME}/kmlc_mnist_train_logistic_model/export/step_1001/
${BUCKET_NAME}/kmlc_mnist_train_logistic_model/export/step_2001/
${BUCKET_NAME}/kmlc_mnist_train_logistic_model/export/step_3001/
```

가장 최근에 저장된 모델을 선택하세요. 우리는 3001번째 step에 저장된 모델을 가지고 진행하도록 하겠습니다:

```
EXPORTED_MODEL_DIR=${BUCKET_NAME}/kmlc_mnist_train_logistic_model/export/step_3001/
```

다음 명령어를 사용하여 Batch Prediction을 시작하세요:

```
JOB_NAME=kmlc_mnist_batch_predict_$(date +%Y%m%d_%H%M%S); \
gcloud ml-engine jobs submit prediction ${JOB_NAME} --verbosity=debug \
--model-dir=${EXPORTED_MODEL_DIR} --data-format=TF_RECORD \
--input-paths='gs://kmlc_test_train_bucket/mnist/test.tfrecords' \
--output-path=${BUCKET_NAME}/batch_predict/${JOB_NAME}.csv --region=us-east1 \
--runtime-version=1.2 --max-worker-count=10
```

이후 진척 상황은 [Google Cloud ML Jobs console](https://console.cloud.google.com/ml/jobs)에서 확인하실 수 있습니다. Job을 좀 더 빠르게 진행하기를 원하시면  'max-worker-count'를 증가시키시면 됩니다.

Batch Prediction Job이 끝나면 다음 명령어를 통해서 결과물을 CSV 형식으로 저장해주세요:

```
# Copy the output of the batch prediction job to a local directory
mkdir -p /tmp/batch_predict/${JOB_NAME}
gsutil -m cp -r ${BUCKET_NAME}/batch_predict/${JOB_NAME}.csv /tmp/batch_predict/${JOB_NAME}.csv
```

/tmp/batch_predict/${JOB_NAME}.csv 를 [Kaggle](https://inclass.kaggle.com/c/mnist-tutorial-machine-learning-challenge)에 제출해주세요.


### Accessing Files on Google Cloud

[Google Cloud storage browser](https://console.cloud.google.com/storage/browser)을 통해서 이전에 생성한 storage bucket을 직접 조회할 수 있습니다. 해당 버킷에 저장된 Trained model, CSV 파일 등을 직접 조회할 수 있습니다.

다른 방법으로는 'gsutil' 명령어를 통해서 파일을 직접 다운로드 받을 수도 있습니다.
예를 들어 방금 생성한 Prediction CSV를 로컬 머신으로 다운로드 받고자 한다면 다음 명령어를 실행하시면 됩니다:

```
gsutil cp $BUCKET_NAME/${JOB_TO_EVAL}/predictions.csv .
```

### Using Larger Machine Types
몇몇 복잡한 모델들은 단일 GPU로 훈련시 모델 수렴까지 일주일 이상이 걸리기도 합니다. 이러한 모델들은 좀 더 많은 GPU가 있는 강력한 머신으로 좀 더 빠르게 훈련이 가능합니다. 4개의 GPU가 있는 머신을 사용하하기 위해서는, '--config' 플래그를 'mnist/cloudml-4gpu.yaml'로 변경해주세요. 이 플래그는 소모하는 Cloud credit 역시 4배로 늘어난다는 점 주의하시기 바랍니다.

## Running on Your Own Machine

### Requirements

시작하기 전 Tensorflow가 준비되어 있어야 합니다. 만약 아직 설치하지 않으셨다면 [tensorflow.org](https://www.tensorflow.org/install/)에 쓰인 설명을 따라주세요.

다음 명령어를 통해 Python 2.7 이상, 그리고 Tensorflow 1.0.1 이상이 설치되어 있는지 확인하시기 바랍니다:

```sh
python --version
python -c 'import tensorflow as tf; print(tf.__version__)'
```

Downloading files
``` sh
gsutil cp gs://kmlc_test_train_bucket/mnist/train* .
gsutil cp gs://kmlc_test_train_bucket/mnist/test* .
gsutil cp gs://kmlc_test_train_bucket/mnist/validation* .
```

Training
```sh
python train.py --train_data_pattern='/path/to/training/files/*' --train_dir=/tmp/mnist_train --model=LogisticModel --start_new_model
```

Validation
```sh
python eval.py --eval_data_pattern='/path/to/validation/files' --train_dir=/tmp/mnist_train --model=LogisticModel --run_once=True
```

Generating submission
```sh
python inference.py --output_file=/path/to/predictions.csv --input_data_pattern='/path/to/test/files/*' --train_dir=/tmp/mnist_train
```

## Overview of Models

예제 코드는 Logistic model의 구현체를 담고 있습니다:

*   `LogisticModel`: Linear projection of the output features into the label
                     space, followed by a sigmoid function to convert logit
                     values to probabilities.

## Overview of Files

### Training
*   `train.py`: 훈련을 위한 전달인자와 과정을 정의합니다. 변경 가능한 전달인자로는 훈련 데이터셋의 위치, 훈련에 사용될 모델, 배치 사이즈, 로스 함수, 학습 레이트 등이 있습니다. 모델에 따라 get_input_data_tensors()를 변경하여 데이터가 셔플되는 과정을 수정하실 수도 있습니다.
*   `losses.py`: 로스 함수를 정의합니다. losses.py에 정의된 어떤 로스 함수도 train.py에서 사용하실 수 있습니다.
*   `models.py`: 모델을 정의하기 위한 Base class를 포함하고 있습니다.
*   `mnist_models.py`: Contains the definition for models that that take the aggregated features as input, and you should add your own models here. You can invoke any model by calling train.py using --model=YourModelName.
*   `mnist_models.py`: 인풋에서 관찰해야 하는 특성들을 입력으로 받을 수 있는 모델에 대한 정의가 있습니다. 여러분은 여러분만의 모델 또한 이 곳에 정의하셔야 합니다. train.py를 호출할 떄 인자로 --model=YourModelName 을 전달하여 여러분의 모델을 부를 수 있습니다.
*   `export_model.py`: 배치 예측 작업에서 쓰일 모델을 추출하기 위한 클래스를 제공합니다.
*   `readers.py`: Contains the definition of the dataset, and describes how input data are prepared. You can preprocess the input files by modifying prepare_serialized_examples(). For example, you may want to resize the data or introduce some random noise.
*   `readers.py`: 데이터에 대한 정의와 인풋 데이터가 어떤 식으로 준비되어야 하는지를 표기하고 있습니다. prepare_serialized_examples()를 변경함으로써 인풋 파일을 프리프로세스 할 수도 있습니다. 예를 들면 데이터를 리사이즈하거나 랜덤 노이즈를 섞을 수 있습니다.

### Evaluation
*   `eval.py`: 모델을 평가하기 위한 기본적인 스크립트입니다. 모델이 훈련되고 나면 --train_dir=/path/to/model 과 --model=YourModelName 인자와 함께 eval.py를 호출하여 train_dir 경로 안에 있는 모델을 불러올 수 있습니다. 보통은 이 파일을 변경할 필요는 없습니다.
*   `eval_util.py`: 모든 종류의 평가 메트릭들을 계산하는 클래스를 제공합니다.
*   `average_precision_calculator.py`: Average precision을 계산하는 함수들을 제공합니다.
*   `mean_average_precision_calculator.py`: Mean average precision을 계산하는 함수들을 제공합니다.

### Inference
*   `inference.py`: Generates an output file containing predictions of the model over a set of data. Call inference.py on the test data to generate a list of predicted labels. For the supervised learning problems, the evaluation is based on the accuracy of the predicted labels.
*   `inference.py`: inference.py를 호출하여 테스트 데이터에 대한 예측 레이블들을 생성하세요. Supervised learning 문제에서 채점은 Predicted label의 정확도에 기반하여 진행됩니다.

### Misc
*   `README.md`: 이 문서입니다.
*   `utils.py`: 일반적인 유틸리티 함수입니다.
*   `convert_prediction_from_json_to_csv.py`: 배치 prediction의 JSON 출력 파일을 CSV로 변경합니다.

## TODO for participants
* mnist_models.py 안에 여러분만의 모델을 생성하세요.
* losses.py 안에 로스 함수를 생성하세요.
* 필요하다면, readers.py 안에 입력 프리프로세싱을 추가해주세요
* train.py 파일 안에서 batch size나 learning rate와 같은 parameter들을 조절하고, 필요하다면 training procedure도 수정해주세요.
* 여러분의 모델 이름과 로스 함수를 이용해서 train.py를 호출하여 모델을 훈련하세요.
* Call eval.py to examine the performance of trained model on validation data.
* eval.py를 호출하여 검증 데이터 위에서 훈련된 모델의 성능을 검증하세요.
* inference.py를 통하여 훈련된 모델을 테스트 데이터에 적용하고, 결과 예측 레이블 정보를 채점을 위해 [Kaggle](https://inclass.kaggle.com/c/mnist-tutorial-machine-learning-challenge)에 제출하세요.
* 본 시나리오는 MLC에 참가하기 위한 가장 쉬운 시나리오 중 하나이며, 여러분만의 솔루션과 더 나은 성능을 위해서라면 어떤 파일을 수정하셔도 무방합니다.

## Etc
* [Tensorflow MNIST Tutorial](https://www.tensorflow.org/get_started/mnist/beginners)

## About This Project
This project is meant help people quickly get started working with the
[mnist](https://inclass.kaggle.com/c/mnist-tutorial-machine-learning-challenge) dataset.
This is not an official Google product.
