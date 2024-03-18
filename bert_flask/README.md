# Running Bert Flask with Intel optimizations on AWS Sagemaker

### Overview

This example demonstrates the deployment of a BERT (Bidirectional Encoder Representations from Transformers) model using Flask on AWS SageMaker with leveraging Intel based optimizations to enhance inference performance and efficiency.

### Prerequisites

1. Launch an EC2 instance with C6i instance type and Ubuntu 20.04 Linux AMI.
2. SSH into the EC2 C6i instance. (All subsequent steps are to be executed from within the EC2 instance.)
3. Git clone the SageMaker samples repository.
   ``` 
   git clone https://github.com/aws-samples/amazon-sagemaker-custom-container.git 
   ```
4. Install Docker. (Refer to AWS CLI documentation for ECR and Docker usage [here](https://docs.aws.amazon.com/AmazonECR/latest/userguide/getting-started-cli.html))
   ``` 
   sudo apt install docker.io 
   ```
5. Install AWS CLI.
   ``` 
   sudo apt install awscli 
   ```

### Instructions

1. Execute the command to create the Docker image that will handle quantization and model generation, pushing it to ECR.
   ``` 
   build_and_push.sh <docker image name> 
   ```
2. Run the Docker image and execute the following command:
   ``` 
   sudo docker run --rm -it -p 8080:8080 -p 8081:8081 -p 8082:8082 -p 7070:7070 -p 7071:7071 bert_large_c6i 
   ```
3. Inside the Docker container, run the provided Python script to generate both FP32 and INT8 models. (This process may vary in duration based on the C6i instance size)
   ``` 
   python quantize_with_ds_ep.py 
   ```
4. The Python script generates model artifacts required for the SageMaker endpoint. This step involves creating and uploading the model tar gzip file to an AWS S3 bucket.
   ``` 
   tar -czf both_bert_model.tar.gz model_int8.pt model_fp32.pt tokenizer.json vocab.txt special_tokens_map.json tokenizer_config.json 

   aws s3 cp both_bert_model.tar.gz s3://intelc6i 
   ```

### Create SageMaker model and endpoint
1. Register the model in Sagemaker: 
```bash
aws sagemaker create-model --model-name c6ibothbert --execution-role-arn "<ROLE ARN>" --primary-container '{
  "ContainerHostname": "BertMainContainer1Hostname",
  "Image": "<AWS ACCOUNT NUMBER>.dkr.ecr.us-east-1.amazonaws.com/bert_dataset_flask:latest",
  "ModelDataUrl": "https://intelc6i.s3.amazonaws.com/bert_dataset_model.tar.gz",
  "Environment" : {
         "SAGEMAKER_CONTAINER_LOG_LEVEL": "20",
         "SAGEMAKER_REGION": "us-east-1"
         }
 }' 
``` 
2. Register the inference endpoint configuration for the endpoint referencing the model name and ec2 instance class.
```bash
aws sagemaker create-endpoint-config --endpoint-config-name c6ibothbert --production-variants '[
{
  "VariantName" : "TestVariant1",
  "ModelName" : "c6ibothbert",
  "InitialInstanceCount" : 1,
  "InstanceType" : "ml.c6i.4xlarge"
 }
]'
```
3. Register the endpoint referencing the endpoint config name.
```
 aws sagemaker create-endpoint --endpoint-name c6ibothbert --endpoint-config-name c6ibothbert 
```
4. Check the status of the endpoint to confirm that it is InService. 
5. To test the FP32 model, execute the following script:
``` python invoke-FP32.py ```
6. To test the INT8 model, execute the following script:
``` python invoke-INT8.py ```
7. Validate the response and check the answer. 
 
Remember to shutdown and cleanup the AWS services and artifacts created as part of this exercise.
Make sure that you save any code or artifacts that you may want for future reference, before you run the cleanup steps. 

### Steps for cleanup
1. Run the following commands to delete the SageMaker model and endpoint. 
``` 
aws sagemaker delete-endpoint --endpoint-name  c6ibothbert
aws sagemaker delete-endpoint-config --endpoing-config-name c6ibothbert 
aws sagemaker delete-model --model-name c6ibothbert
```
2. Delete the S3 objects created during this exercise.
3. Delete the IAM roles and policies created during this exercise.
4. Delete the docker image from ECR
```
aws ecr batch-delete-image \
      --repository-name <image name> \
      --image-ids imageTag=<tag> \
      --region <AWS region>
```