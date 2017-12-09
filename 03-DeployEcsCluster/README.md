# Running an ECS Cluster

**Quick jump:**

* [Tutorial overview](https://github.com/bemer/lts-workshop/tree/master/02-CreatingDockerImage#tutorial-overview)
* [Creating your first image](https://github.com/bemer/lts-workshop/tree/master/02-CreatingDockerImage#creating-your-first-image)
* [Setting up the IAM Roles](https://github.com/bemer/lts-workshop/tree/master/02-CreatingDockerImage#setting-up-the-iam-roles)
* [Configuring the AWS CLI](https://github.com/bemer/lts-workshop/tree/master/02-CreatingDockerImage#configuring-the-aws-cli)
* [Creating the Container registries with ECR](https://github.com/bemer/lts-workshop/tree/master/02-CreatingDockerImage#creating-the-container-registries-with-ecr)
* [Pushing our tested images to ECR](https://github.com/bemer/lts-workshop/tree/master/02-CreatingDockerImage#pushing-our-tested-images-to-ecr)

## Tutorial overview

This tutorial will guide you through the creation of an AWS ECS Cluster and the deployment of a container with a simple Python application created in the previous lab.

In order to run this tutorial, you must have completed the following steps:

* [Setup Environment](https://github.com/bemer/lts-workshop/tree/master/01-SetupEnvironment)
* [Creating your Docker image](https://github.com/bemer/lts-workshop/tree/master/02-CreatingDockerImage)


## Creating the Cluster

Once you've signed into your AWS account, navigate to the [ECS console](https://console.aws.amazon.com/ecs/home?region=us-east-1#/clusters). This URL will redirect you to the ECS interface. If this is your fist time using this service, you will see the "*Clusters*" screen without any clusters in it:

![clusters screen](https://github.com/bemer/lts-workshop/blob/master/03-DeployEcsCluster/images/clusters_screen.png)

Let's them create our first ECS Cluster. Click in the button **Create cluster** and in the following screen select the **EC2 Linux + Networking** cluster template:

![cluster template](https://github.com/bemer/lts-workshop/blob/master/03-DeployEcsCluster/images/cluster_template.png)

You will them be asked to input information about your new cluster. Fill the fields in the *Configure cluster* screen and note that here you can create a new VPC or select a existing one. We recommend you to create a new VPC using the wizard, so it is going to be easy clean up your account after the event.

Remember to select your keypair, so you will be able to access the EC2 instances in your cluster in order to perform any kind of troubleshooting later.

Note that this wizard is also going to create a security group for you allowing access in the port 80 (TCP).

Click in **Create**.

When the creation process finishes, you will see the following screen:

![cluster created](https://github.com/bemer/lts-workshop/blob/master/03-DeployEcsCluster/images/cluster_created.png)

You can them click in the button **View Cluster** to see your cluster.

## Creating the ALB

Now that we've created our cluster, we need an Application Load Balancer (ALB)[https://aws.amazon.com/elasticloadbalancing/applicationloadbalancer/] to route traffic to our endpoints. Compared to a traditional load balancer, an ALB lets you direct traffic between different endpoints.  In our example, we'll use the enpoint:  `/app`.

To create the ALB:

Navigate to the **EC2 Service Console**, and select **Load Balancers** from the left-hand menu.  Choose **Create Load Balancer**:

![choose ALB](https://github.com/bemer/lts-workshop/blob/master/03-DeployEcsCluster/images/select_alb.png)

Name your ALB **lts-workshop** and add an HTTP listener on port 80:

![name ALB](https://github.com/bemer/lts-workshop/blob/master/03-DeployEcsCluster/images/create_alb.png)

> In a production environment, you should also have a secure listener on port 443.  This will require an SSL certificate, which can be obtained from [AWS Certificate Manager](https://aws.amazon.com/certificate-manager/), or from your registrar/any CA.  For the purposes of this demo, we will only create the insecure HTTP listener. DO NOT RUN THIS IN PRODUCTION.

Next, select your VPC and add at least two subnets for high availability.  Make sure to choose the VPC that we created with the ECS wizard.  If you have multiple VPC, and you're not sure which VPC is the correct one, you can find its ID from the VPC console.

![add VPC](https://github.com/bemer/lts-workshop/blob/master/03-DeployEcsCluster/images/configure_alb.png)

Next, add a security group.  If you ran the ECS first run wizard, you should have an existing group called something like **EC2ContainerService-lts-workshop-EcsSecurityGroup**.  If you don't have this, check you've chosen the correct VPC, as security groups are VPC specific.  If you still don't have this, you can create a new security groups with the following rule:

    Ports	    Protocol	    Source
     80	          tcp	       0.0.0.0/0

Choose the security group, and continue to the next step:  adding routing.  For this initial setup, we're just adding a dummy healthcheck on `/`.  We'll add specific healthchecks for our service endpoint when we register it with the ALB.

![add routing](https://github.com/bemer/lts-workshop/blob/master/03-DeployEcsCluster/images/configure_alb_routing.png)

Finally, skip the "Register targets" step, and continue to review. If your values look correct, click **Create**.

Note:  If you created your own security group, and only added a rule for port 80, you'll need to add one more.  Select your security group from the list > **Inbound** > **Edit** and add a rule to allow your ALB to access the port range for ECS (0-65535).  The final rules should look like:

     Type        Ports        Protocol        Source
     HTTP          80	        tcp	         0.0.0.0/0
     All TCP      0-65535       tcp       <id of this security group>


## Create your Task Definition

Before you can register a container to a service, it needs be a part of a Task Definition. Task Definitions define things like environment variables, the container image you wish to use, and the resources you want to allocate to the service (port, memory, CPU).  To create a Task Definition, choose **Task Definitions** from the ECS console menu.  Then, choose **Create new Task Definition**. Select EC2 as the *Launch type compatibility*:

![type compatibility](https://github.com/bemer/lts-workshop/blob/master/03-DeployEcsCluster/images/task_compatibility.png)

And them click in **Next Step**. Name your task **lts-demo-app**:

![create task def](https://github.com/bemer/lts-workshop/blob/master/03-DeployEcsCluster/images/create_task_def.png)

Continue to adding a container definition. Click in **Add container**. Now we will add the informations about the image that we created in the previous tutorial. The name of our container will be `lts-demo-app`. In the *Image* field, we need to add the URI from our repository. You can get this URI in the *Repositories* screen, by selecting the repository *lts-demo-app* created before.

Add `128` in the *Memory Limits* field and in *Port mapping* add `0` in the *Host port* field and `3000` in the *Container port*.

![container def](https://github.com/bemer/lts-workshop/blob/master/03-DeployEcsCluster/images/container_def.png)


A few things to note here:

- We've specified a specific container image, including the `:latest` tag.  Although it's not important for this demo, in a production environment where you were creating Task Definitions programmatically from a CI/CD pipeline, Task Definitions could include a specific SHA, or a more accurate tag.

- Under **Port Mappings**, we've specified a **Container Port** (3000), but left **Host Port** as 0.  This was intentional, and is used to facilitate dynamic port allocation.  This means that we don't need to map the Container Port to a specific Host Port in our Container Definition-  instead, we can let the ALB allocate a port during task placement.  To learn more about port allocation, check out the [ECS documentation here](http://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_PortMapping.html).

Once you've specified your Port Mappings, scroll down and add a log driver.  There are a few options here, but for this demo, select **Auto-configure CloudWatch Logs**:

![aws log driver](https://github.com/bemer/lts-workshop/blob/master/03-DeployEcsCluster/images/setup_logdriver.png)

Once you've added your log driver, add the Container Definition, and create the Task Definition.


## Create your Services

Navigate back to the ECS console, and choose the cluster that you created during the first run wizard.  This should be named **lts-workshop**.  If you don't have a cluster named **lts-workshop**, create one following the procedures in  [Creating the cluster](https://github.com/bemer/lts-workshop/tree/master/03-DeployEcsCluster#creating-the-cluster).

Next, you'll need to create your app service.  From the cluster detail page, choose **Services** > **Create**.

Select `EC2` as the *Launch Type* choose the Task Definition you created in the previous section. For the purposes of this demo, we'll only start one copy of this task.  In a production environment, you will always want more than one copy of each task running for reliability and availability.

You can keep the default **AZ Balanced Spread** for the Task Placement Policy.  To learn more about the different Task Placement Policies, see the [documentation](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-placement-strategies.html), or this [blog post](https://aws.amazon.com/blogs/compute/introducing-amazon-ecs-task-placement-policies/).

![create service](https://github.com/bemer/lts-workshop/blob/master/03-DeployEcsCluster/images/create_service.png)

Click in **Next**.

Under **Load balancing**, select **Application Load Balancer**. Under *Container to load balance*, select the container "*lts-workshop*" and click in **Add to load balancer**:

![add to ALB](https://github.com/bemer/lts-workshop/blob/master/03-DeployEcsCluster/images/add_container_to_alb.png)

This final step allows you to configure the container with the ALB.  When we created our ALB, we only added a listener for HTTP:80.  Select this from the dropdown as the value for **Listener**.  For **Target Group Name**, enter a value that will make sense to you later, like **lts-demo-app**.  For **Path Pattern**, the value should be **`/app*`**.  This is the route that we specified in our Python application. In the **Evaluation order**, add the number `1`.

Finally, **Health check path**, use the value `/app`.

![configure container ALB](https://github.com/bemer/lts-workshop/blob/master/03-DeployEcsCluster/images/configure_container_alb.png)

If the values look correct, click **Next Step**.

Since we will not use Auto Scaling in this tutorial, in the **Set Auto Scaling** screen, just click in **Next Step** and after reviewing your configurations, click in **Create Service**.


## Testing our service deployments from the console and the ALB

You can see service level events from the ECS console.  This includes deployment events. You can test that of your service deployed, and registered properly with the ALB by looking at the service's **Events** tab:

![deployment event](https://github.com/bemer/lts-workshop/blob/master/03-DeployEcsCluster/images/steady_state_service.png)

We can also test from the ALB itself.  To find the DNS A record for your ALB, navigate to the EC2 Console > **Load Balancers** > **Select your Load Balancer**.  Under **Description**, you can find details about your ALB, including a section for **DNS Name**.  You can enter this value in your browser, and append the endpoint of your service, to see your ALB and ECS Cluster in action:

![alb web test](https://github.com/abby-fuller/ecs-demo/blob/master/images/alb_app_response.png)

You can see that the ALB routes traffic appropriately based on the path we specified when we registered the container:  `/app*/` requests go to our app service.


## More in-depth logging with Cloudwatch

When we created our Container Definitions, we also added the awslogs driver, which sends logs to [Cloudwatch](https://aws.amazon.com/cloudwatch/).  You can see more details logs for your services by going to the Cloudwatch console, and selecting first our log group:

![log group](https://github.com/abby-fuller/ecs-demo/blob/master/images/loggroups.png)

And then choosing an individual stream:

![event streams](https://github.com/abby-fuller/ecs-demo/blob/master/images/event_streams.png)

## That's a wrap!

Congratulations!  You've deployed an ECS Cluster with two working endpoints.  