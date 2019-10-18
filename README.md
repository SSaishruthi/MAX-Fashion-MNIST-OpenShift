[![Build Status](https://travis-ci.org/SSaishruthi/max-fashion-mnist.svg?branch=master)](https://travis-ci.org/SSaishruthi/max-fashion-mnist)

# MAX-Fashion-MNIST

#### Classify fashion and clothing items. 

![mnist](samples/data.png)

# Data Source: 

IBM Developer - [Data Asset Exchange](https://developer.ibm.com/exchanges/data/)

Curated free and open datasets under open data licenses for enterprise data science.

Link to download: https://developer.ibm.com/exchanges/data/all/fashion-mnist/

# Framework

The model is developed using the Tensorflow framework.

# Labels
Each training and test example is assigned to one of the following labels:

| Label | Description |
| --- | --- |
| 0 | T-shirt/top |
| 1 | Trouser |
| 2 | Pullover |
| 3 | Dress |
| 4 | Coat |
| 5 | Sandal |
| 6 | Shirt |
| 7 | Sneaker |
| 8 | Bag |
| 9 | Ankle boot |

# Requirements

1. Docker
2. Python IDE or code editors
3. Pre-trained model weights stored in a downloadable location
4. List of required python packages
5. Input pre-processing code
6. Prediction/Inference code
7. Output variables
8. Output post-processing code

# Steps

1. [Fork the Template and Clone the Repository](#fork-the-template-and-clone-the-repository)
2. [Update Dockerfile](#Update-dockerfile)
3. [Update Package Requirements](#update-package-requirements)
4. [Update API and Model Metadata](#update-api-and-model-metadata)
5. [Update Scripts](#update-scripts)
6. [Build the model Docker image](#build-the-model-docker-image)
7. [Run the model server](#run-the-model-server)

## Fork the Template and Clone the Repository

1. Login to GitHub and go to [MAX Skeleton](https://github.com/IBM/MAX-Skeleton)

2. Click on `Use this template` and provide a name for the repo.

3. Clone the newly created repository using the below command:

```bash
$ git clone https://github.com/......
```

## Update Dockerfile

Update,

- `ARG model_bucket=` with the link to the model file public storage that can be downloaded

- `ARG model_file=` with the model file name. 
   
For testing purpose, update as below:

`model_bucket = https://max-assets-dev.s3.us-south.cloud-object-storage.appdomain.cloud/max-demo/1.0.0`

`model_file = assets.tar.gz`

## Update Package Requirements

Add required python packages for running the model prediction to requirements.txt. 

Following packages are required for this model:

   - numpy==1.14.1
   - Pillow==5.4.1
   - h5py==2.9.0
   - tensorflow==1.14
   

## Update API and Model Metadata

1. In `config.py`, update the API metadata.

   - API_TITLE 
   - API_DESC 
   - API_VERSION 

2. Set `MODEL_PATH = 'asssets/fashion_mnist.h5'`

   _NOTE_: Model files are always downloaded to `assets` folder inside docker.

3. In `core/model.py`, fill in the `MODEL_META_DATA` 
       
   - Model id
   - Model name (e.g. MAX-Fashion-MNIST)
   - Description of the model (e.g. Classify clothing and fashion items)
   - Model type based on what the model does (e.g. Digit recognition)
   - Source to the model belongs
   - Model license (e.g. Apache 2.0)
   
## Update Scripts

All you need to start wrapping your model is pre-processing, prediction and post-processing code.
  
1. In `core/model.py`, load the model under the `__init__()` method of the `ModelWrapper` class. 
  Here, the saved model (`.h5` format) can be loaded using the command below.
  
    ```python
    global sess
    global graph
    sess = tf.Session() 
    graph = tf.get_default_graph()
    set_session(sess)
    self.model = tf.keras.models.load_model(path)
   ```

2. In `core/model.py`, input pre-processing functions should be placed under the `_pre_process` function.
   Here, the input image needs to be read and converted into an array of acceptable shape.
  
   ```python
   # Open the input image
   img = Image.open(io.BytesIO(inp))
   print('reading image..', img.size)
   # Convert the PIL image instance into numpy array and
   # get in proper dimension.
   image = tf.keras.preprocessing.image.img_to_array(img)
   print('image array shape..', image.shape)
   image = np.expand_dims(image, axis=0)
   print('image array shape..', image.shape)
   ```
 
3. Following pre-processing, we will feed the input to the model. Place the inference code under the `_predict` method in `core/model.py`. The model will return a list of class probabilities, corresponding to the likelihood of the input image to belong to respective class. There are 10 classes (digit 0 to 9), so `predict_result` will contain 10 values.
  
   ```python
   with graph.as_default():
      set_session(sess)
      predict_result = self.model.predict(x)
      return predict_result
   ```
     
4. Following inference, a post-processing step is needed to reformat the output of the `_predict` method. It's important to make sense of the results before returning the output to the user. Any post-processing code will go under the `_post_process` method in `core/model.py`.

    In order to make sense of the predicted class digits, we will add the `CLASS_DIGIT_TO_LABEL` variable to the `config.py` file. This will serve as a mapping between class digits and labels to make the output more understandable to the user. 
    
   ```python
   CLASS_DIGIT_TO_LABEL = {
   0: "T-shirt/top",
   1: "Trouser",
   2: "Pullover",
   3: "Dress",
   4: "Coat",
   5: "Sandal",
   6: "Shirt",
   7: "Sneaker",
   8: "Bag",
   9: "Ankle boot"
   }
   ```

   We will import this map at the top of the `model.py` file.

   ```python
   from config import DEFAULT_MODEL_PATH, CLASS_DIGIT_TO_LABEL
   ```
  
   The class with the highest probability will be assigned to the input image. Here, we will use our imported `CLASS_DIGIT_TO_LABEL` variable to map the class digit to the corresponding label.
   
   
   ```python
   # Extract prediction probability using `amax` and
   # digit prediction using `argmax`
   return [{'probability': np.amax(result),
            'prediction': CLASS_DIGIT_TO_LABEL[np.argmax(result)]}]
   ```
   
5. The predicted class and it's probability are the expected output. In order to return these values in the API, we need to add these two fields to `label_prediction` in `api/predict.py`.
  
   ```python
   label_prediction = MAX_API.model('LabelPrediction', {
   'prediction': fields.String(required=True),
   'probability': fields.Float(required=True)
   })
   ```
   
   In addition, the output response has two fields `status` and `predictions` need to be updated as follows. 
  
   ```python
   predict_response = MAX_API.model('ModelPredictResponse', {
     'status': fields.String(required=True, description='Response status message'),
     'predictions': fields.List(fields.Nested(label_prediction), description='Predicted labels and probabilities')
   })
   ```
 
   _NOTE_: These fields can vary depending on the model.
   

6. Finally, assign the output from the post-processing step to the appropriate response field in `api/predict.py` to link the processed model output to the API.

   ```python
   # Assign result
   result['predictions'] = preds
   ```

## Build the model Docker image

To build the docker image locally, run:

```
$ docker build -t max-mnist .
```

If you want to print debugging messages make sure to set `DEBUG=True` in `config.py`.

## Run the model server

To run the docker image, which automatically starts the model serving API, run:

```
$ docker run -it -p 5000:5000 max-mnist
```

## (Optional) Update Test Script

1. Add test images to the `samples/` directory. In our implementation, we have added a picture of the `T-shirt/top` category and named it `1.jpeg`.

2. Add a few integration tests using pytest in tests/test.py to verify that your model works. 

   Example:

   - Update model endpoint and sample input file path.

      ```python
      model_endpoint = 'http://localhost:5000/model/predict'
      file_path = 'samples/1.jpeg'
      ```

   - Check if the prediction is `T-shirt/top`.

      ```python
      assert response['predictions'][0]['prediction'] == "T-shirt/top"
      ```

3. To enable Travis CI testing uncomment the docker commands and pytest command in `.travis.yml`.

# (Optional) Deploy to production with Kubernetes

## Requirements
- Docker
- an [IBM Cloud account](https://ibm.biz/BdzAHm)
- the [IBM Cloud command-line tools](https://cloud.ibm.com/docs/cli?topic=cloud-cli-getting-started), including `kubectl`
- (recommended) basic knowledge on Kubernetes (e.g. [this blogpost](https://medium.com/google-cloud/kubernetes-101-pods-nodes-containers-and-clusters-c1509e409e16))

## Instructions

1. Log in to IBM Cloud and create a (free) Kubernetes cluster
   
    Find the [cluster in the catalog](https://cloud.ibm.com/kubernetes/catalog/cluster) and follow the steps to create a Kubernetes cluster.

2. Access the Kubernetes cluster
   
    Following the creation of the cluster, it takes a couple minutes for the cluster to finalize setting up. Use this time to set up the `ibmcloud` CLI to access the cluster. Instructions, including how to download and install the CLI, are found in the `Access` tab of the dashboard.

    The instructions include:

    - Logging in to the cluster using the CLI
    - Downloading the kubeconfig files for your cluster
    - Setting the KUBECONFIG environment variable

3. Upload your Docker image to DockerHub
   
    Your Kubernetes cluster will need to download the container image of your model in order to run it. Log in to [DockerHub](https://hub.docker.com/) and upload the image to your account.

    The link to your image will have the following format:

    `[account-username]/[image-name]:[version-tag]`

    If you have your image built locally, you can upload it using the Docker command line tools.

    ```bash
    docker login
    ```
    ```bash
    docker push [account-username]/[image-name]:[version-tag]
    ```

    Replace the `[account-username]` with your Docker username, `[image-name]` with the name of the image you built, and `[version-tag]` with how you want to label the current version of the image (e.g. `v1`).

4. Deploy

    Spinning up our application is typically accomplished in three steps.
    - Create a Namespace
    - Create a Service
    - Create a Deployment

    Every step in the configuration of the cluster is achieved by applying a configuration file (also called `configmap`) to the cluster. These configmaps are formatted in the YAML markup language, for which templates are abundantly present online. We can design  separate YAML files for every aspect of the deployment, or we can stitch them together in one big YAML file.

    - Namespace
  
      ```yaml
      # Define a namespace
      apiVersion: v1
      kind: Namespace
      metadata:
        name: [NAMESPACE-NAME]
      ```
    
    - Service
  
      ```yaml
      # Define a service in the namespace
      apiVersion: v1
      kind: Service
      metadata:
        name: [MODEL-NAME]
        namespace: [NAMESPACE-NAME]
      spec:
        selector:
          app: [MODEL-NAME]
        ports:
        # the exposed port in our Dockerfile
        - port: 5000
        type: NodePort
      ```

    - Deployment

      ```yaml
      # Define a deployment
      apiVersion: extensions/v1beta1
      kind: Deployment
      metadata:
        name: [MODEL-NAME]
        namespace: [NAMESPACE-NAME]
        labels:
          app: [MODEL-NAME]
      spec:
        selector:
          matchLabels:
            app: [MODEL-NAME]
        # the number of replicas
        replicas: 2
        template:
          metadata:
            labels:
              app: [MODEL-NAME]
          spec:
            containers:
            - name: [MODEL-NAME]
              # the Docker image on DockerHub
              image: [DOCKER-ACCOUNT-USERNAME]/[IMAGE-NAME]:[VERSION-TAG]
              imagePullPolicy: Always
              # the exposed container port
              ports:
              - containerPort: 5000
              env:
              - name: CORS_ENABLE
                value: 'true'
      ```
    
    These three components have been stitched together into one file (`kubernetes_template.yaml` in this repository).
    Replace the `[NAMESPACE-NAME]`, `[MODEL-NAME]`, `[DOCKER-ACCOUNT-USERNAME]`, `[IMAGE-NAME]`, and `[VERSION-TAG]` placeholder variables with their actual values. Save the file, and uses the `kubectl` CLI to configure this map.

    ```bash
    kubectl create -f kubernetes_template.yaml
    ```

    (optional) Use the following commands to do a couple status checks. Your deployment should be up and running.

    ```bash
    # show namespaces
    kubectl get namespaces
    # show services
    kubectl get services -n [NAMESPACE-NAME]
    # show deployments
    kubectl get deployment [MODEL-NAME] -n [NAMESPACE-NAME]
    # show nodes
    kubectl get nodes -o wide
    ```

    _NOTE: This is a template configuration. Once you get comfortable with spinning up containers on Kubernetes, you will need to adjust the configmap for your specific use-case._

5. Access your application

    - Note down the external IP address of your node (value for `EXTERNAL-IP`)

        ```bash
        kubectl get nodes -o wide
        ```

    - Note down the exposed port (the number under `PORT(S)` in the `30000-32767` range)
  
        ```bash
        kubectl get svc -n [NAMESPACE-NAME]
        ```

    Now, combine these two into the following format:

    `http://[EXTERNAL-IP]:[PORT-NUMBER]`

    Navigate there and you should see the API of your application. Congratulations, you have completed the harest part!

    _To learn more about scaling this cluster and advanced configuration such as LoadBalancers or Ingress, feel free to ask us about more advanced resources._

## More deployment-related resources

- [Deploy MAX models to IBM Cloud with Kubernetes](https://developer.ibm.com/tutorials/deploy-max-models-to-ibm-cloud-with-kubernetes/)
- [Kubernetes 101 Medium blogpost](https://medium.com/google-cloud/kubernetes-101-pods-nodes-containers-and-clusters-c1509e409e16)
- [Linode.com container deployment tutorial](https://www.linode.com/docs/applications/containers/kubernetes/deploy-container-image-to-kubernetes/)
