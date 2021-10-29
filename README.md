Deploy sentiment-light from Github to AWS ECS
CAVEAT: Not working yet
---
This repo demonstrates how to take the sentiment-light app located in my repo and push it to AWS
The CloudFormation templates are based on this 
[AWS Tutorial](https://github.com/awslabs/ecs-refarch-continuous-deployment). 

### Sections
1. First Things First
2. Modifications to the AWS Example Templates
3. Running this system of Templates to Deploy 

### First Things First
In [Github](https://github.com), make sure to 
[create a personal access token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token).

### Modifications to the AWS Example Templates
##### Parameter Values
I added a few Parameters to the parent template to feed into the child templates:
ImageTag, ContainerPort, DesiredCount, and HostPort. Make sure to declare any Parameters you add to the 
corresponding Resource's Parameter block. Also, declare them in the Parameters section of the 
child template. You need to do both these things so the values make it through the Stack build. 

##### Buildspec
The next thing I did was replace the BuildSpec phases in the `templates/deployment-pipeline.yml.`
I could have pointed the `CodeBuildProject::Properties::Buildspec` to a buildspec.yml in the repo, 
but I wanted all the deployment code in this one place. I also added the EnvironmentVariables 
needed to fill in the variables in the BuildSpec.

##### TaskDefinition
Next, in the `templates/service.yaml`, we have to modify the TaskDefinition. I pointed the Image 
pushed in the Buildspec. I sent it to ECR, but you can easily modify the code to push to DockerHub.

##### ECRRepository
I needed to create an ECRRepository to push the image to. That resource is in the `templates/service.yaml`.

##### Port Numbers
In `templates/service.yaml` in the `FargateService` and `EC2Service` block, the `LoadBalancer` ports
need to match the standard port 5000 that Flask apps usually run on. In the 
`TaskDefinition::ContainerDefinitions::PortMappings`, the ContainerPort should be 5000 also. The HostPort,
needs to handle both the Fargate and EC2 deployments, so I put in a conditional `!If [ Fargate, 5000, 5001 ]`.
This maps the host port to 5000 for fargate deployments and 5001 for EC2. 

I created Parameters for the HostPort and ContainerPort in the parent template to feed into the 
`templates/service.yaml` to make things easier for your deployment.  


### Running this system of Templates to Deploy 
run:<br>
`./bin/deploy continuous-deployment-sentiment-light`
