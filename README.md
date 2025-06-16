# Deploying a Flask API

This is the project starter repo for the course Server Deployment, Containerization, and Testing.

In this project you will containerize and deploy a Flask API to a Kubernetes cluster using Docker, AWS EKS, CodePipeline, and CodeBuild.

The Flask app that will be used for this project consists of a simple API with three endpoints:

- `GET '/'`: This is a simple health check, which returns the response 'Healthy'. 
- `POST '/auth'`: This takes a email and password as json arguments and returns a JWT based on a custom secret.
- `GET '/contents'`: This requires a valid JWT, and returns the un-encrpyted contents of that token. 

The app relies on a secret set as the environment variable `JWT_SECRET` to produce a JWT. The built-in Flask server is adequate for local development, but not production, so you will be using the production-ready [Gunicorn](https://gunicorn.org/) server when deploying the app.



## Prerequisites

* Docker Desktop - Installation instructions for all OSes can be found <a href="https://docs.docker.com/install/" target="_blank">here</a>.
* Git: <a href="https://git-scm.com/downloads" target="_blank">Download and install Git</a> for your system. 
* Code editor: You can <a href="https://code.visualstudio.com/download" target="_blank">download and install VS code</a> here.
* AWS Account
* Python version between 3.7 and 3.9. Check the current version using:
```bash
#  Mac/Linux/Windows 
python --version
```
You can download a specific release version from <a href="https://www.python.org/downloads/" target="_blank">here</a>.

* Python package manager - PIP 19.x or higher. PIP is already installed in Python 3 >=3.4 downloaded from python.org . However, you can upgrade to a specific version, say 20.2.3, using the command:
```bash
#  Mac/Linux/Windows Check the current version
pip --version
# Mac/Linux
pip install --upgrade pip==20.2.3
# Windows
python -m pip install --upgrade pip==20.2.3
```
* Terminal
   * Mac/Linux users can use the default terminal.
   * Windows users can use either the GitBash terminal or WSL. 
* Command line utilities:
  * AWS CLI installed and configured using the `aws configure` command. Another important configuration is the region. Do not use the us-east-1 because the cluster creation may fails mostly in us-east-1. Let's change the default region to:
  ```bash
  aws configure set region us-east-2  
  ```
  Ensure to create all your resources in a single region. 
  * EKSCTL installed in your system. Follow the instructions [available here](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html#installing-eksctl) or <a href="https://eksctl.io/introduction/#installation" target="_blank">here</a> to download and install `eksctl` utility. 
  * The KUBECTL installed in your system. Installation instructions for kubectl can be found <a href="https://kubernetes.io/docs/tasks/tools/install-kubectl/" target="_blank">here</a>. 


## Initial setup

1. Fork the <a href="https://github.com/udacity/cd0157-Server-Deployment-and-Containerization" target="_blank">Server and Deployment Containerization Github repo</a> to your Github account.
1. Locally clone your forked version to begin working on the project.
```bash
git clone https://github.com/SudKul/cd0157-Server-Deployment-and-Containerization.git
cd cd0157-Server-Deployment-and-Containerization/
```
1. These are the files relevant for the current project:
```bash
.
├── Dockerfile 
├── README.md
├── aws-auth-patch.yml #ToDo
├── buildspec.yml      #ToDo
├── ci-cd-codepipeline.cfn.yml #ToDo
├── iam-role-policy.json  #ToDo
├── main.py
├── requirements.txt
├── simple_jwt_api.yml
├── test_main.py  #ToDo
└── trust.json     #ToDo 
```

     
## Project Steps

Completing the project involves several steps:

1. Write a Dockerfile for a simple Flask API
2. Build and test the container locally
3. Create an EKS cluster
4. Store a secret using AWS Parameter Store
5. Create a CodePipeline pipeline triggered by GitHub checkins
6. Create a CodeBuild stage which will build, test, and deploy your code

For more detail about each of these steps, see the project lesson.


## Running the App Locally

1. Install python dependencies

 `pip install -r requirements.txt`

2. Set up the environment

```bash
export JWT_SECRET='myjwtsecret' 
export LOG_LEVEL=DEBUG
```

In Windows 
```bash
set JWT_SECRET='myjwtsecret' 
set LOG_LEVEL=DEBUG
```

3. Run the app

`
python main.py
`

4. Install a command-line JSON processor

```
# For Linux
sudo apt-get install jq  
# For Mac
brew install jq 
# For Windows, 
chocolatey install jq
```


5. Access endpoint /auth

```bash
export TOKEN=`curl --data '{"email":"abc@xyz.com","password":"mypwd"}' --header "Content-Type: application/json" -X POST localhost:8080/auth  | jq -r '.token'`
echo $TOKEN
```

In Windows

```bash
set TOKEN=`curl --data '{"email":"abc@xyz.com","password":"mypwd"}' --header "Content-Type: application/json" -X POST localhost:8080/auth  | jq -r '.token'`
echo %TOKEN%
```
6. Access endpoint /contents

```bash
curl --request GET 'http://localhost:8080/contents' -H "Authorization: Bearer ${TOKEN}" | jq .

```
In Windows
```bash
curl --request GET 'http://localhost:8080/contents' -H "Authorization: Bearer %TOKEN%" | jq .

```

## I.b. Containerizing and Running Locally

1. Verify the Dockerfile

```
# Use the `python:3.7` as a source image from the Amazon ECR Public Gallery
# We are not using `python:3.7.2-slim` from Dockerhub because it has put a  pull rate limit. 
FROM public.ecr.aws/sam/build-python3.7:latest
# Set up an app directory for your code
COPY . /app
WORKDIR /app
# Install `pip` and needed Python packages from `requirements.txt`
RUN pip install --upgrade pip
RUN pip install -r requirements.txt
# Define an entrypoint which will run the main app using the Gunicorn WSGI server.
ENTRYPOINT ["gunicorn", "-b", ":8080", "main:APP"]
```

2. Store Environment Variables

Containers cannot read the values stored in your localhost's environment variables. Therefore, create a file named `.env_file` and save both `JWT_SECRET` and `LOG_LEVEL` into that `.env_file`. We will use this file while creating the container. Here, we do not need the export command, just an equals sign:

```
JWT_SECRET='myjwtsecret'
LOG_LEVEL=DEBUG
```

3. Start the Docker Desktop service.
4. Build an image

```
docker build -t myimage .
# Other useful commands
# Check the list of images
docker image ls
# Remove any image
docker image rm <image_id>
```

5. Create and run a container

```bash
docker run --name myContainer --env-file=.env_file -p 80:8080 myimage
# Other useful commands
# List running containers
docker container ls
docker ps
# Stop a container
docker container stop <container_id>
# Remove a container
docker container rm <container_id>
```

6. Check the endpoints

```bash
# Flask server running inside a container
curl --request GET 'http://localhost:80/'
# Flask server running locally (only the port number is different)
curl --request GET 'http://localhost:8080/'
# Calls the endpoint 'localhost:80/auth' with the email/password as the message body. 
# The return JWT token assigned to the environment variable 'TOKEN' 
export TOKEN=`curl --data '{"email":"abc@xyz.com","password":"WindowsPwd"}' --header "Content-Type: application/json" -X POST localhost:80/auth  | jq -r '.token'`
echo $TOKEN
# Decrypt the token and returns its content
curl --request GET 'http://localhost:80/contents' -H "Authorization: Bearer ${TOKEN}" | jq .
```

## II.a. Overview of the CD Pipeline

1. Create an EKS Cluster, IAM Role for CodeBuild, and Authorize the CodeBuild
   1. Create an EKS Cluster - Start with creating an EKS cluster in your preferred region, using `eksctl` command.

   2. IAM Role for CodeBuild - Create an IAM role that the Codebuild will assume to access your k8s/EKS cluster. This IAM role will have the necessary access permissions (attached JSON policies),

   3. Authorize the CodeBuild using EKS RBAC - Add IAM Role to the Kubernetes cluster's configMap.

2. Deployment to Kubernetes using CodePipeline and CodeBuild
   1. Generate a Github access token <br>
   Cenerate an access-token from your Github account. We will share this token with the Codebuild service so that it can listen to the the repository commits.

   2. Create Codebuild and CodePipeline resources using CloudFormation template <br>
   Create a pipeline watching for commits to your Github repository using Cloudformation template (.yaml) file.

   3. Set a Secret using AWS Parameter Store  <br>
   In order to pass your JWT secret to the app in Kubernetes securely, you will set the JWT secret using AWS parameter store.

   4. Build and deploy <br>
   Finally, you will trigger the build based on a Github commit.

## II.b. Create an EKS Cluster and IAM Role

### Prerequisite
You must have the following:

1. AWS CLI installed and configured using the `aws configure` command.

2. The EKSCTL and KUBECTL command-line utilities installed in your system. Check and note down the KUBECTL version, using:

```
kubectl version
```

3. You current working directory must be:

```
cd cd0157-Server-Deployment-and-Containerization
```

1. Create an EKS (Kubernetes) Cluster

  Create - Create an EKS cluster named “simple-jwt-api” in a region of your choice:

```bash
eksctl create cluster --name simple-jwt-api --nodes=2 --version=1.32 --instance-types=t2.medium --region=us-east-1

```

2. Verify - After creating the cluster, check the health of your clusters nodes:

```bash
kubectl get nodes
```

3. Delete when the project is over

Remember, in case you wish to delete the cluster, you can do it using eksctl:

```bash
eksctl delete cluster simple-jwt-api  --region=us-east-2
```

###  Create an IAM Role for CodeBuild

1. Get your AWS account id::

```bash
aws sts get-caller-identity --query Account --output text
```

2. Update the `trust.json` file with your AWS account id.

```json
{
"Version": "2012-10-17",
"Statement": [
    {
        "Effect": "Allow",
        "Principal": {
            "AWS": "arn:aws:iam::<ACCOUNT_ID>:root"
        },
        "Action": "sts:AssumeRole"
    }
]
}
```

Replace the <ACCOUNT_ID> with your actual AWS account ID.

3. Create a role, 'UdacityFlaskDeployCBKubectlRole', using the trust.json trust relationship:

```bash
aws iam create-role --role-name UdacityFlaskDeployCBKubectlRole --assume-role-policy-document file://trust.json --output text --query 'Role.Arn'

```

4. Policy is also a JSON file where we will define the set of permissible actions that the Codebuild can perform. <br>
We have given you a policy file, iam-role-policy.json(opens in a new tab), containing the following permissible actions: "eks:Describe*" and "ssm:GetParameters".

```json
{
 "Version": "2012-10-17",
 "Statement": [
     {
         "Effect": "Allow",
         "Action": [
             "eks:Describe*",
           	 "ssm:GetParameters"
         ],
         "Resource": "*"
     }
 ]
}
```

5. Attach the iam-role-policy.json policy to the 'UdacityFlaskDeployCBKubectlRole' as:

```bash
aws iam put-role-policy --role-name UdacityFlaskDeployCBKubectlRole --policy-name eks-describe --policy-document file://iam-role-policy.json

```

### Authorize the CodeBuild using EKS RBAC

1. Fetch - Get the current configmap and save it to a file:

```bash
# Mac/Linux - The file will be created at `/System/Volumes/Data/private/tmp/aws-auth-patch.yml` path
kubectl get -n kube-system configmap/aws-auth -o yaml > /tmp/aws-auth-patch.yml
# Windows - The file will be created in the current working directory
kubectl get -n kube-system configmap/aws-auth -o yaml > aws-auth-patch.yml
```

2. Edit - Open the aws-auth-patch.yml file using any editor, such as VS code editor:

```bash
# Mac/Linux
code /System/Volumes/Data/private/tmp/aws-auth-patch.yml
# Windows
code aws-auth-patch.yml
```

Add the following group in the data → mapRoles section of this file. YAML is indentation-sensitive, therefore refer to the snapshot below for a correct indentation:

```
   - groups:
       - system:masters
     rolearn: arn:aws:iam::<ACCOUNT_ID>:role/UdacityFlaskDeployCBKubectlRole
     username: build   
```

3. Update - Update your cluster's configmap:
 
```bash
# Mac/Linux
kubectl patch configmap/aws-auth -n kube-system --patch "$(cat /tmp/aws-auth-patch.yml)"
# Windows
kubectl patch configmap/aws-auth -n kube-system --patch "$(cat aws-auth-patch.yml)"
```

## II.c. Deployment to Kubernetes using CodePipeline and CodeBuild

1. Generate a Github access token <br>
A Github access token will allow Codebuild to monitor when a repo is changed. A token is analogous to your Github password and can be generated [here](https://github.com/settings/tokens/). You should generate the token with full control of private repositories, as shown in the image below. Be sure to save the token somewhere that is secure.

2. Create Codebuild and CodePipeline resources using CloudFormation template

   1. Modify the template <br>
   There is a file named `ci-cd-codepipeline.cfn.yml` provided in your starter repo. This is the template file that you will use to create your CodePipeline pipeline and CodeBuild project Open this file, and go to the 'Parameters' section. These are parameters that will accept values when you create a stack. Ensure that the following values are used for the parameter variables:

![test](https://video.udacity-data.com/topher/2022/May/627cc180_screenshot-2022-05-11-at-1.44.39-pm/screenshot-2022-05-11-at-1.44.39-pm.jpeg)


3. Create Stack
Use the AWS web-console to create a stack for CodePipeline using the CloudFormation template file ci-cd-codepipeline.cfn.yml. Go to the CloudFormation service(opens in a new tab) in the AWS console. Press the Create Stack button.

4. Save a Secret in AWS Parameter Store

```bash
aws ssm put-parameter --name JWT_SECRET --overwrite --value "myjwtsecret" --type SecureString
## Verify
aws ssm get-parameter --name JWT_SECRET
```

Once you submit your project and receive the reviews, you can consider deleting the variable from parameter-store using:
```bash
aws ssm delete-parameter --name JWT_SECRET
```

### The Working Project

1. **Push a commit** - To check if the pipeline works, Make a git push to your repository to trigger an automatic build as:

```bash
## Verify the remote destination. 
## It should point to the repo in your account (not the repo in the Udacity account). 
## Otherwise, FORK the Udacity repo, and then clone it locally
git remote -v
## Make the changes locally
git status
## Add the changed file to the Git staging area
git add <filename>
## Provide a meaningful commit description
git commit -m “my comment“
## Push to the local master branch to the remote master branch
git push
```

2. Verify - In the AWS console go to the CodePipeline dashboard(opens in a new tab). You should see that the build is running, and it should succeed.
3. Test your Endpoint - To test your API endpoints, get the external IP for your service:
```bash
kubectl get services simple-jwt-api -o wide
export TOKEN=`curl -d '{"email":"<EMAIL>","password":"<PASSWORD>"}' -H "Content-Type: application/json" -X POST <EXTERNAL-IP URL>/auth  | jq -r '.token'`
curl --request GET '<EXTERNAL-IP URL>/contents' -H "Authorization: Bearer ${TOKEN}" | jq 
```
 


