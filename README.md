## Setup
#Clone the repository
git clone https://github.com/Vennilavanguvi/Brain-Tasks-App.git
cd Brain-Tasks-App
#create Dockerfile                  FROM nginx:alpine
                                    COPY dist/ /usr/share/nginx/html
                                    EXPOSE 3000
                                    CMD ["nginx", "-g", "daemon off;"]
docker build -t app3 .
docker run -d -p 3000:80 app3

Host port : 3000
Container : 80

Verify http://localhost:3000 in browser



#Create ECR repo
Amazon ECR > Private registry > Repositories > ns/app3

aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin [ACCOUNT_ID].dkr.ecr.ap-south-1.amazonaws.com
docker tag app3:latest [ACCOUNT_ID].dkr.ecr.ap-south-1.amazonaws.com/ns/app3:latest
docker push [ACCOUNT_ID].dkr.ecr.ap-south-1.amazonaws.com/ns/app3:latest


#Create cluster in AWS 
Amazon Elastic Kubernetes Service > Clusters > funny-pop-rainbow

# deployment.yml


apiVersion: apps/v1
kind: Deployment
metadata:
  name: brain-tasks-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: brain-task
  template:
    metadata:
      labels:
        app: brain-task
    spec:
      containers:
      - name: brain-tasks-container
        image: 404198732694.dkr.ecr.ap-south-1.amazonaws.com/ns/app3:latest
        ports:
        - containerPort: 80


# service.yml

apiVersion: v1
kind: Service
metadata:
  name: brain-task-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
    service.beta.kubernetes.io/aws-load-balancer-type: "classic"
spec:
  type: LoadBalancer
  selector:
    app: brain-task
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80


kubectl apply -f deployment.yml  
kubectl apply -f service.yml

#deployment.yml points to your ECR repository, Kubernetes will pull the image from ECR and run it in the pods 
#service.yml will Creates a Kubernetes Service & Exposes pods via LoadBalancer

C:\Users\cinderlla\Brain-Tasks-App>kubectl get svc
NAME                 TYPE           CLUSTER-IP      EXTERNAL-IP                                                                     PORT(S)        AGE
brain-task-service   LoadBalancer   10.100.249.87   k8s-default-braintas-f4c61ad7ac-387358f25cae6e7a.elb.ap-south-1.amazonaws.com   80:31154/TCP   35h

#Verify External IP in browser

#Created repo in gitHub > https://github.com/kestersam30-stack/Project3

#Created code build
Developer Tools > CodeBuild > Build projects > brain-tasks-build
 
# buildspec.yml
version: 0.2

env:
  variables:
    AWS_DEFAULT_REGION: "ap-south-1"
    REPOSITORY_URI: "404198732694.dkr.ecr.ap-south-1.amazonaws.com/ns/app3"
    IMAGE_TAG: "latest"
    CLUSTER_NAME: "funny-pop-rainbow"

phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin 404198732694.dkr.ecr.ap-south-1.amazonaws.com

  build:
    commands:
      - echo Build started on `date`
      - docker build -t $REPOSITORY_URI:$IMAGE_TAG .

  post_build:
    commands:
      - echo Pushing Docker image
      - docker push $REPOSITORY_URI:$IMAGE_TAG

      - echo Updating kubeconfig
      - aws eks update-kubeconfig --region $AWS_DEFAULT_REGION --name $CLUSTER_NAME

      - echo Deploying to EKS
      - kubectl apply -f deployment.yaml
      - kubectl apply -f service.yaml

artifacts:
  files:
    - '**/*'


The buildspec.yml file defines the build steps executed by AWS CodeBuild. It authenticates with Amazon ECR, builds the Docker image, pushes the image to ECR, and deploys the updated container to the EKS cluster using kubectl.

#Create Pipeline 
Developer Tools > CodePipeline > Pipelines > Brain-task-PL

#Logs are also viewed




#Purpose: 
Stores the application code and configuration files.
#Process:
Developer pushes code changes to GitHub.
CodePipeline monitors the repository.
When a commit is pushed, the pipeline automatically triggers.
#Result:
Pipeline execution starts automatically.
