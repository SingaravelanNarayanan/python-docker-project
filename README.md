project---2
Creating a full-fledged Python web application, Dockerizing it, storing the Docker image in AWS Elastic Container Registry (ECR), and deploying it to AWS Elastic Container Service (ECS) and Elastic Kubernetes Service (EKS) involves several steps. Here’s a step-by-step guide:

Step 1: Create a Python Web Application
Let's create a simple Flask web application.

1---app.py

from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello():
    return "Hello, World!"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
	
2---requirements.txt

Flask==2.1.2

Step 2: Create a Dockerfile
Dockerfile

3--dockerfile

# Use an official Python runtime as a parent image
FROM python:3.9-slim

# Set the working directory in the container
WORKDIR /app

# Copy the current directory contents into the container at /app
COPY . /app

# Install any needed packages specified in requirements.txt
RUN pip install --no-cache-dir -r requirements.txt

# Make port 5000 available to the world outside this container
EXPOSE 5000

# Define environment variable
ENV NAME World

# Run app.py when the container launches
CMD ["python", "app.py"]

Step 3: Build and Push Docker Image to ECR
First, configure AWS CLI and authenticate Docker to your ECR registry:

4--
aws configure
5--
aws ecr get-login-password --region <your-region> | docker login --username AWS --password-stdin <your-account-id>.dkr.ecr.<your-region>.amazonaws.com

6--Create an ECR repository:

aws ecr create-repository --repository-name flask-web-app --region <your-region>

7--Build and push the Docker image to ECR:

docker build -t flask-web-app .
docker tag flask-web-app:latest <your-account-id>.dkr.ecr.<your-region>.amazonaws.com/flask-web-app:latest
docker push <your-account-id>.dkr.ecr.<your-region>.amazonaws.com/flask-web-app:latest

Step 4: Deploy to ECS
ecs-task-definition.json

{
  "family": "flask-web-app",
  "networkMode": "awsvpc",
  "containerDefinitions": [
    {
      "name": "flask-web-app",
      "image": "<your-account-id>.dkr.ecr.<your-region>.amazonaws.com/flask-web-app:latest",
      "essential": true,
      "portMappings": [
        {
          "containerPort": 5000,
          "hostPort": 5000
        }
      ]
    }
  ],
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512"
}
8--Create the ECS task definition:

aws ecs register-task-definition --cli-input-json file://ecs-task-definition.json
9--Create an ECS cluster:

aws ecs create-cluster --cluster-name flask-web-app-cluster
10--Run the ECS service:

aws ecs create-service --cluster flask-web-app-cluster --service-name flask-web-app-service --task-definition flask-web-app --desired-count 1 --launch-type FARGATE --network-configuration "awsvpcConfiguration={subnets=[<subnet-id>],securityGroups=[<security-group-id>]}"
Step 5: Deploy to EKS
k8s-deployment.yaml


apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-web-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: flask-web-app
  template:
    metadata:
      labels:
        app: flask-web-app
    spec:
      containers:
      - name: flask-web-app
        image: <your-account-id>.dkr.ecr.<your-region>.amazonaws.com/flask-web-app:latest
        ports:
        - containerPort: 5000
---
apiVersion: v1
kind: Service
metadata:
  name: flask-web-app
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 5000
  selector:
    app: flask-web-app
10--Apply the deployment to EKS:


kubectl apply -f k8s-deployment.yaml

Step 6: Summary
Create Python Web Application: Write the Flask application.
Dockerize: Create a Dockerfile and build the image.
Push to ECR: Authenticate with ECR, create a repository, and push the Docker image.
Deploy to ECS: Register the task definition, create a cluster, and deploy the service.
Deploy to EKS: Write Kubernetes deployment YAML and apply it to your EKS cluster.
Ensure you have the necessary permissions and correct configurations for AWS CLI, ECR, ECS, and EKS. This is a simplified guide, and real-world applications might require more detailed configurations and error handling.
