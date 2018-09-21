# AWS EB Deployment Instructions

by J.R. Ruggiero
Additional Notes by David Cervantes

### Intro notes (Disclaimer lol)

By no means do I believe this is the optimal way to deploy, these are just the steps I took to deploy our multi-container docker app.

The following guide was the most helpful for me in setting this up. Most of the steps are copied from this guide:

[Multi-Docker EB deployment guide](http://blog.digitopia.com/elastic-beanstalk-docker-deployment/)

My file structure is as follows:

#### Root

- docker-compose.yml: Docker compose instructions. I believe this file is not actually used in the deployment of the app, but you probably have this file here and its ok.
- Dockerrun.aws.json: This is the file used to build our instance. We will get to this file

#### Sub folders

I have two folders, one holding the client side angular application and the other running a server on node.

#### Containers

When composing locally, I run three containers: One for the client, one for the server, and a postgres image. There is no folder for the database because we aren't doing anything in the database in regards to building it or running an entrypoint script, it is just the image. That will come in to play later.

### SERVER .env handling

When deploying to AWS you will have to change how your .env is processed. Because of this, it is recommended that if you have to make the following changes, you either do so in a seperate branch, or conform your app to the following steps:
- Remove all quotations from your .env
- Inside the start method for the package.json, you will have to add an env command. My script looks like this: "start": "rimraf dist && gulp start && env $(cat .env) node --max-old-space-size=5000 dist/main.js". It is the env $(cat .env) that is new
- Comment out the requirement of the env services (envfile and dotenv) inside your api.ts file.

### CLI Requirements

You will need to install both the AWS CLI and the EB CLI. Both CLIs will also require Python to be installed as it utilizes the pip command.

[AWS CLI installation instructions](https://docs.aws.amazon.com/cli/latest/userguide/installing.html)



[EB CLI Installation instruction](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/eb-cli3-install.html)


### Step 1: Prepare your images

The first step will be uploading the images into a docker image repository onto AWS.

- Go to the Elastic Container Service. On the left side there should be a tag for repositories. You should create one for the client and the server. We do not need one for the database because it is the base image only.

- When you create the repo and you click on it, it will give you instructions for pushing an image. We will be following these instructions. You can view this page later by clicking on the 'View Push Commands' button

- Open your docker terminal if you are on Windows. I believe Mac's have the docker machine initialized automatically so you can go to a normal terminal.
Run the aws cli command listed on the push instructions:
`$(aws ecr get-login --no-include-email --region us-west-2)`

There is a differnet command listed for Windows users, but I was able to run the above command as long as my docker terminal was up.
The result of this command should be a large docker login string. That is the next command to run; copy that line, paste it into your terminal and run it.

- You will need to build the container for the respective repo with the following command:
`docker build -t <your-container-production-name> <directory-of-the-container>`
For example, my command was *➜  ~ docker build -t countability-client noble-one-client-multi/ 

// for today:       docker build -t >>>>>countability-tnr-sandbox-client>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>> noble-one-client-multi
                    docker build -t >>>>>countability-tnr-sandbox-server>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>> noble-one-server-multi
                    docker build -t >>>>>countability-tnr-sandbox-client>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>> noble-one-client-multi
**NOTE:** Keep track of the name you are giving your container, you will need it later
**FURTHER BUILD NOTE** If you are using volumes to mount your local files to mirror the files on the container, you will need to adjust your build because AWS will not have access to files on your computer. In my case, this required adding a line to the *Dockerfile* that copies the contents of my folder into the container. If you choose this process as well, I would also recommend adding a **.dockerignore** to avoid copying your node modules, since those will be instantiated upon build.


- Tag the container you just build for upload:
`docker tag <your-container-name-you-just-built>:latest <address-of-the-repo-in-the-push-commands>/<container-name>:latest`

- This links the container in a similar way that setting the remote for git works. Now you can push the image:
`docker push <repo-address-in-push-commands>/<container-name>:latest`
The push process will take a few mimnutes.

When you finish, you should see your image on the repository. Repeat the process for all containers that require building. Make sure each container has its own repository.

### Step 2: Create the Dockerrun.aws.json file

This file will have similar functionality to a docker-compose. 

[Dockerrun MultiContainer Docker Configuration Guide](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/create_deploy_docker_v2config.html)

My Example\

(I removed the repo location and replaces with <REPOSDDRESS>)

```
{
    "AWSEBDockerrunVersion": 2,
    "containerDefinitions": [
        {
            "essential": true,
            "image": "postgres:latest",
            "name": "postgres-database",
            "memoryReservation": 256,
            "portMappings": [
                {
                    "containerPort": 5432,
                    "hostPort": 5434
                }
            ]
        },
        {
            "name": "countability-server",
            "image": "<REPOADDRESS>/countability-server:latest",
            "essential": true,
            "memoryReservation": 2048,
            "mountPoints": [
               
            ],
            "portMappings": [
                {
                    "containerPort": 3000,
                    "hostPort": 3000
                },
                {
                    "containerPort": 3001,
                    "hostPort": 3001
                },
                {
                    "containerPort": 3002,
                    "hostPort": 3002
                },
                {
                    "containerPort": 3003,
                    "hostPort": 3003
                },
                {
                    "containerPort": 3004,
                    "hostPort": 3004
                },
                {
                    "containerPort": 3005,
                    "hostPort": 3005
                }
            ],
            "links": [
                "postgres-database"
            ]
        },
        {
            "name": "countability-client",
            "image": "<REPOADDRESS>/countability-client:latest",
            "essential": true,
            "memoryReservation": 2048,
            "mountPoints": [
                
            ],
            "portMappings": [
                {
                    "containerPort": 4200,
                    "hostPort": 80
                },
                {
                    "containerPort": 49153,
                    "hostPort": 49153
                }
            ],
            "links": [
                "countability-server",
                "postgres-database"
            ]
        }
    ],
    "volumes": [
        {
            "host": {
                "sourcePath": "./noble-one-client-multi"
            },
            "name": "_Countability-Client-Multi"
        },
        {
            "host": {
                "sourcePath": "/home/node/client/node_modules"
            },
            "name": "_Countability-Client-NodeModules"
        },
        {
            "host": {
                "sourcePath": "./noble-one-server-multi"
            },
            "name": "_Countability-Server-Multi"
        },
        {
            "host": {
                "sourcePath": "/home/node/server/node_modules"
            },
            "name": "_Countability-Server-NodeModules"
        }
    ]
}
```

Some highlights:

- How you assign your memory is very important. Failure to assign enough memory to a container will cause it to crash either during start-up or immediately after. Try to utilize a soft limit with `memoryReservation` as opposed to a hard limit, and keep in mind the instance will need a small bit of memory to operate on its own. You can also try running your containers locally and use `docker stats` too see how much memory your containers are using.
- The image will link to the images in your repository
- Be sure to redirect port 80 to your front end to avoid having to enter a port number when visiting the application page.
- It's a good idea to keep the container name equal to the name you gave the container when you tagged them.

[AWS instance types](https://aws.amazon.com/ec2/instance-types/)

### Step 3: Run `eb init`

Make sure you do so in the root directory. You will be asked for the app name.

### Step 4: Test locally with `eb local`

To test your deployment locally, run `eb local run`. This will bind to your current docker daemon, so if you are on windows and you want to browse to your app, it is on whatever the result of `docker-machine ip` (Usually 192.168.99.100). Linux and Mac will find the app on localhost.

You can also use `eb local status` in a seperate terminal to check the status of your local deployment. `docker ps` also relays some information. Pushing Control-C in the terminal that ran `eb local run` will stop the deployment, but it may not stop all the containers from running. You may have to shut them down with the command line or a program like Kitematic.

### Step 5: Make sure your EB role has access to the repository

To do this, you will need to search services for *IAM*. Under IAM resources click on *Roles: #* and find `aws-elasticbeanstalk-ec2-role` and click on it. Under the permissions tab, you will need to click the `Attach` button and apply the `AmazonEC2ContainerRegistryReadOnly` policy. This gives EB access to the images you just pushed.

### Step 6: git add/commit your Dockerrun.aws.json file
e
Or else it will not read it.

### Step 7: Create your environment

Using the `eb create <environment-name> <options>` command you can now build an environment that will utilize the instructions in your Dockerrun.aws.json file. Here are some of the option highlights

`-i <container-type i.e. t2.medium>` - Failure to include this automatically assigns you a t2.micro, which will probably not have enough memory for your app.  ie. `eb create -i t2.medium`
`--single` - Loads your app as a single instance instead of having multiple images manages by a load balancer. Single is good for testing, but not for deployment.  (not used for cucrm)
`--elb-type <load-balancer-type` specifies the type of loadbalancer to use. (We used `network`)
`-k <keyname>` - Links an EC2 key pair you've created to use when attemptingto SSH into your instance (We used `cucumber-key`)


[eb create command detail](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/eb3-create.html)

### Step 8: Open up any necessary ports in your environment with the Security Group

Creating your environment should also create a security group for your environment. You can access it by selecting *EC2* under services, and then click on the link that shows how many security groups you have up.

You should see one security group with the name of the environment you clicked on. Under description, it should say 'SecurityGroup for ElasticBeanstalk environment'. Click the check box and some information should populate down below. Click on inbound.

You will probably only see ports for base http (80) and ssh (22). If your app requires any more ports to be access, be sure to add them here. Simply exposing them on the container level is not enough. (Make to add additonal ports for server (ie. Server: Custom TCP with Ports 3000-3010, and source anywhere; Database same as server except with ports 5432-5434))


### Step 9: Does it work

You should be able to find your environment under *Elastic BEanstalk* from the services menu. Click on the environment you just created. There should be a url just under the title. If you redirected your front end to port 80, your app should open. But you may need to make some changes to your app.

### Step 10: Make any changes you need to IP related tasks in your app

For example, my app has a CORS whitelist, and no address can access it without being on the list. Unfortuntely, we don't have our address until we've created our environment. Adjust any variables that need it and re-build, tag, ad push your images (i.e. Step 1). You will need to restart your environment after pushing

### MISC ISSUES

For whatever reason, while the url on the EB dashboard works on the site for the client, it does not work for the server... which is weird. You can, however, use the IP assigned to that instance. You can find this IP going to Elastic Container Service, selecting the cluster that contains the application, select the environment, click on the *ECS Instances* tab, and click on the container instance. You can also click on the task at the bottom and find the location of each container.

Update: Now you can add changes to the CLIENT_URL='http://countability.bwrd.io' to the env.ts file (in server),  and similar url changes in environment.ts (in client).

Also make sure to update the IP Address in Route 53.  Search for Route 53, then under 'DNS Management' option on the Dashboard, click on 'Hosted Zones', then click on bwrd.io, then select countability.bwrd.io, and change the IPv4 address to what it should be.  The most recent IP address can be found in the Elastic Container Service.  Just follow the directions in the first paragraph of this section.

Random Notes

Old IP Adddress for Countability: 34.219.225.231
New IP Address: 34.219.206.204