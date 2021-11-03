Deploy sentiment-light from Github to AWS ECS
---
This repo demonstrates how to take the sentiment-light app located in my repo and push it to a Farget or EC2 
ECS deployment. The CloudFormation templates are based on this 
[AWS Tutorial](https://github.com/awslabs/ecs-refarch-continuous-deployment). I've added Parameters to 
make it easier to modify the repo to other apps. I've also added some details of the changes I had to make from 
the tutorial repo above to get the app deployed. 

### Sections
1. First Things First
2. Modifications to the AWS Example Templates
3. Running this system of Templates to Deploy 

### First Things First
In [Github](https://github.com), make sure to 
[create a personal access token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token).
You'll need this when creating the Stack.

There is also a dependency in the DeploymentPipeline on the Service. 
This means that the DeploymentPipeline waits on the Service to be built, but the Task in the Service is looking 
for the image. So, the Stack just hangs up in creating the Service. So it is necessary, to create an ECR Repository and 
[push Docker image of the repo to ECR](https://docs.aws.amazon.com/AmazonECR/latest/userguide/getting-started-cli.html).
Doing this initially will keep your Stack from hanging when creating the Service.

### Modifications to the AWS Example Templates
##### Parameter Values
I added a few Parameters to the parent template to feed into the child templates:
ImageTag, ContainerPort, DesiredCount, HostPort, etc. I added the Parameters that I thought could generalize to 
apps with different languages. Make sure to declare any Parameters you add to the 
corresponding Resource's Parameter block in . Also, declare them in the Parameters section of the 
child template. You need to do both these things so the values make it through the Stack build. 

##### Buildspec
The next thing I did was replace the BuildSpec phases in the `templates/deployment-pipeline.yml.`
I could have pointed the CodeBuildProject::Properties::Buildspec to a buildspec.yml in the repo, 
but I wanted all the deployment code in this one place. I also added the EnvironmentVariables 
needed to fill in the variables in the BuildSpec.

##### TaskDefinition
Next, in the `templates/service.yaml`, we have to modify the TaskDefinition. I pointed the Image 
pushed in the Buildspec. I sent it to ECR, but you can easily modify the code to push to DockerHub.


##### Port Numbers
Instead of hard-coding port numbers in, the ContainerPort and HostPort Parameters are added to make it easier 
to deploy apps that run on various ports. HostPort is the port number that your service will map to on the host 
if LaunchType is EC2. This needs to be the same as the ContainerPort for Fargate deployments. There are conditionals 
added just make sure this is the case, e.g: `FromPort: !If [ EC2, !Ref HostPort, !Ref ContainerPort ]`

##### Health Checks
My app was slow to boot up because it downloads the sentiment model which takes a bit. So none of the Tasks would 
register as healthy while that was happening, and my app got stuck in an infinite loop of registering and unregistering.
I had to increase the HealthCheckTimeoutSeconds and HealthCheckIntervalSeconds to account for the longer start up time. 
I added these as Parameters for easy deployment modifications.   

### Running this system of Templates to Deploy 
You'll need to send the nested templates to the specified S3 bucket so CloudFormation will be able to get then when 
deploying the Stacks. From the base directory run:

`./bin/deploy continuous-deployment-sentiment-light`

After those files have been delivered to your bucket, go to CloudFormation console in your AWS account. Use the parent
template, named `continuous-deployment-sentiment-light.yaml` in this repo to build the stacks. Check your Parameter values
and proceed on to deploy the Stack. 