+++
title = "Artifactory"
chapter = false
weight = 1
+++

## Artifactory
### Expected Outcome:
* Understanding of a binary repository manager
* Deploy JFrog Artifactory

**Lab Requirements:**
None

**Average Lab Time:** 45-60 minutes

### Introduction
In this module, we will deploy the OpenSource version of https://jfrog.com/artifactory/[JFrog Artifactory] using the managed ECS Fargate service, to your AWS account using a CloudFormation template.

Before we get started lets first change to the module directory.
```
cd ~/environment/aws-modernization-workshop/modules/artifactory
```

#### Getting Started with JFrog Artifactory
Steps to Deploy Artifactory using a CloudFormation template:

##### Launch the Cloudformation template
```
aws cloudformation create-stack --stack-name "artifactory-oss" \
  --template-body=file://artifactory-oss.yml \
  --parameters ParameterKey=ClusterName,ParameterValue="artifactory-oss" \
  --capabilities CAPABILITY_IAM
```

Wait for the CloudFormation template to successfully deploy.
```
until [[ `aws cloudformation describe-stacks \
  --stack-name "artifactory-oss" \
  --query "Stacks[0].[StackStatus]" \
  --output text` == "CREATE_COMPLETE" ]]; \
  do  echo "The stack is NOT in a state of CREATE_COMPLETE at `date`"; \
  sleep 30; done && echo "The Stack is built at `date` - Please proceed"
```

##### Get the container IP
In a production environment, you will want to front your ECS service using an Amazon Application LoadBalancer, but for the purpose of this lab, we will simply connect directly to the container public IP address.
```
aws ec2 describe-network-interfaces \
  --network-interface-ids=$(aws ecs describe-tasks --cluster=artifactory-oss \
    --tasks=`aws ecs list-tasks \
    --cluster=artifactory-oss \
    --query taskArns[0] \
    --output=text` \
    --query tasks[0].attachments[0].details[1].value \
    --output=text) \
  --query NetworkInterfaces[0].Association.PublicIp \
  --output=text
```

#### Log into Artifactory and set up a new admin password.
Open up your browser to http://<ip-from-previous-step>:8081. For example: http://54.202.45.157:8081. You should be presented with a window that looks similar to the one below.

#### Configuring JFrog Artifactory
Steps to Configure Artifactory for the first time:

**Step 1:** You should see a *Welcome to JFrog Artifactory!* welcome screen like below. Click *Next*.

![artifactory](../../images/artifactory-01.PNG)

**Step 2:** Provide a super secure password for your instance of Artifactory. Click *Create*.

![artifactory](../../images/artifactory-02.PNG)

**Step 3:** We will *_NOT_* configure a proxy server for our lab environment. Click *Skip*.

![artifactory](../../images/artifactory-03.PNG)

**Step 4:** Now, we will create our first repository. Select *Maven* from the list. Click *Create*.

![artifactory](../../images/artifactory-04.PNG)

**Step 5:** You have successfully configured Artifactory and should see a message like this. Click *Finish*.

![artifactory](../../images/artifactory-05.PNG)


**NOTE:** *Additional information can be found in the https://www.jfrog.com/confluence/display/RTF/Welcome+to+Artifactory[JFrog Artifactory User Guide]*

#### Configure the Maven repositories.
For the purpose of this lab, we will only be simulating the process of preemptively reviewing libraries as discussed in our Security discussion. So in order for our build process to succeed, we also need to add some upstream repositories to Artifactory.

**Step 1:** Open the Admin Interface on the left and click on "Remote Repositories"

![artifactory](../../images/artifactory-12.png)

**Step 2:** Click on the *+ New* button in the top right corner, and select *Maven* from the Package Type dialog that opens. In the new form that opens fill in the fields like in the image below. Then click *Test*. When the test succeeds, click *Save & Finish* in the bottom right.

![artifactory](../../images/artifactory-13.JPG)

**Step 3:** We now need to edit the virtual repository, to include the newly added remote repository. On the left side menu, open the admin panel again and select *Virtual* under the repositories section. Select the *libs-release* repository. A new window like the one below should open.

![artifactory](../../images/artifactory-14.JPG)

**Step 4:** Move the *primefaces* repository from the *available repositories list* to the *Selected repositories list* by clicking on *primefaces* and then using the green *>* button. Now click *Save and Finish*

#### Modify our Maven Container Build
Now that we have our Artifactory repositories correctly configured, we need to modify the maven settings for our application and have it pull the libraries from the secured repo. We do this by editing the settings.xml file for maven.

We have a pre-written `settings.xml` for you, but we need to replace some of the info inside it, with info specific to your deployment.

**Step 1:** We need to get the public IP from the artifactory container again. This time, we will also store it as an Environment Variable.
```
ART_IP=$(aws ec2 describe-network-interfaces \
  --network-interface-ids=$(aws ecs describe-tasks \
  --cluster=artifactory-oss --tasks=`aws ecs list-tasks \
  --cluster=artifactory-oss --query taskArns[0] --output=text` \
  --query tasks[0].attachments[0].details[1].value --output=text) \
  --query NetworkInterfaces[0].Association.PublicIp --output=text)
```

**Step 2:** Add the IP to our settings.xml
```
sed -i "s/<artifact-ip>/$ART_IP/" settings.xml
```

**Step 3:** Make some modifications to Dockerfile
Now that we have the repository information saved in the `settings.xml` for maven, we also need to make sure that Docker copies the file into the new build environment. We do that by simply adding a single line to the existing `Dockerfile`
```
COPY ./settings.xml /root/.m2/
```

To save some time, we have already done this for you on line #8. We just need you to copy the `settings.xml` and `Dockerfile` into the container app directory.
```
cp {settings.xml,Dockerfile} \
~/environment/aws-modernization-workshop/modules/containerize-application/
```

Your `Dockerfile` should look like:
```
FROM maven:3.5-jdk-7 AS build

# set the working directory
WORKDIR /usr/src/app

# copy the POM and Maven Settings
COPY ./app/pom.xml /usr/src/app/pom.xml
COPY ./settings.xml /root/.m2/

# just install the dependencies for caching
RUN mvn dependency:go-offline
```

**Step 4:**
Now that we have reconfigured our Docker containers we need to rebuild these images.
```
cd ~/environment/aws-modernization-workshop/modules/containerize-application
```

Now that we are back in the *Containerize Application* folder we can rerun
`docker-compose build`

```
docker-compose build petstore
```

With the container rebuilt using the Artifactory repositories we're ready to move on to the next module.
