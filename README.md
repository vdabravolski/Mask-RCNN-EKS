# Train and deploy Mask R-CNN model using Sagemaker Operator for Kubernetes
AWS recently introduced [Sagemaker Operator for Kubernetes](https://aws.amazon.com/blogs/machine-learning/introducing-amazon-sagemaker-operators-for-kubernetes/) which allows development teams to integrate [Sagemaker services](https://aws.amazon.com/sagemaker/) with existing Kubernetes infrastructure.

## Scope
This project provides a set of scripts to build container with Mask R-CNN model, then train and deploy model using Amazon EKS cluster.

We use [TensorPack Mask/Faster-RCNN](https://github.com/tensorpack/tensorpack/tree/master/examples/FasterRCNN) model implementation. Model is trained on [COCO 2017 dataset](http://cocodataset.org/#home).

## Prerequisites
1. You need to have AWS account ([reference](https://aws.amazon.com/premiumsupport/knowledge-center/create-and-activate-aws-account/)). It's recommended to create all AWS resource in specific AWS region (e.g. us-west-2 or us-east-2).
2. You need to have EKS cluster with Sagemaker Operator configured ([setup reference](https://sagemaker.readthedocs.io/en/stable/amazon_sagemaker_operators_for_kubernetes.html#setup-and-operator-deployment))
3. Ensure your AWS account has appropriate resource limits. You'll need at least 2 `ml.p3.16xlarge` instances for Mask R-CNN training. It's recommended to have 4 instances. 
4. (recommended) Use Sagemaker managed Jupyter/JupyterLab notebooks. Note, that your Sagemaker notebook needs to have IAM role which is authorized to access EKS cluster, have `eksctl`, and `kubctl` utils configured. See configuration steps on Step #2.
5. Create S3 bucket to store training data and training output ([reference](https://docs.aws.amazon.com/AmazonS3/latest/user-guide/create-bucket.html)).

## Training Faster R-CNN model
1. Run `./prepare_s3_bucket.sh <YOUR_S3_BUCKET>`. This will upload training dataset to your S3 bucket. 
2. Run `./build_and_push <YOUR_AWS_REGION>`. This will create Mask R-CNN image with training script and push it to AWS ECR. This operation will take 5-10 minutes to complete. If successful, you should see URI of your image. You'll need it later.
3. Update `train.yaml` as follows:
- update "name" field with unique value. This will be a name of your Sagemaker training job;
- update "trainingImage" with URI of your container image (Prepare your envrionment - Step #3);
- update "roleArn" with your Sagemaker execution role ([reference](https://docs.aws.amazon.com/sagemaker/latest/dg/sagemaker-roles.html));
- update "region" with you AWS region;
- update your "S3OutputPath" and "inputDataPath" with your S3 bucket.
4. Run `kubectl apply -f train.yaml` in terminal. This will schedule the Sagemaker job.
5. Monitor your job in Sagemaker console or by running `kubectl describe trainingjob`.

## Hosting trained model
1. Update `hosting.yaml` as follows:
 - "name" with unique endpoint name;
 - "region" with you AWS region;
 - "modelName" with name of already trained model;
 - "roleArn" with your Sagemaker execution role ([reference](https://docs.aws.amazon.com/sagemaker/latest/dg/sagemaker-roles.html));
- "modelDataURL" with link to trained model;
- "image" with image URI in format like \<container registry url\>\/\<image\>:\<version\>.
2. Run `kubectl apply -f hosintg.yaml`. This will schedule endpoint creation.
3. Run `kubectl describe hostingdeployment` or check Sagemaker console for status of endpoint creation.

## Credits
Mask-RCNN training script and docker image are copied from [this AWS repository](https://github.com/awslabs/amazon-sagemaker-examples/tree/master/advanced_functionality/distributed_tensorflow_mask_rcnn)
