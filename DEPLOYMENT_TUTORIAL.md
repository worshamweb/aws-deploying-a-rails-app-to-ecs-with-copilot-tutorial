# AWS Copilot Tutorial: Deploying Serverless Containers to ECS Fargate

This tutorial will guide you through deploying a Rails weather application to Amazon ECS Fargate using AWS Copilot. You'll learn how to deploy serverless containers without managing servers - AWS handles all the infrastructure automatically. The application provides weather forecasts using Google Places autocomplete and WeatherAPI.com integration.

## 🔰 First Time User? Start Here!

**Never used AWS before?** No problem! This tutorial is designed for beginners.

**What you'll need**:
- About 1 hour of time
- A computer with internet access
- An AWS account (free to create)
- Basic command line familiarity (we'll guide you through each command)

**What you'll learn**:
- How to deploy serverless containers to AWS Fargate
- AWS Copilot fundamentals (the easiest way to use ECS)
- Container orchestration without server management
- Modern DevOps practices with Infrastructure as Code
- Cost management for serverless containers

**Don't worry about**:
- Breaking anything (we use safe, isolated resources)
- Server management (Fargate is serverless - AWS handles everything)
- Complex configurations (Copilot handles the complexity)
- High costs (small Fargate containers cost ~$9/month, easily cleaned up)

💡 **Pro tip**: Make sure you've completed all the prerequisite verification steps in the [README](README.md) before starting this deployment tutorial.

## 🚀 Step-by-Step Deployment Guide

> **Before you start**: Make sure you've completed all [prerequisite verification steps](README.md#prerequisites) if you haven't already. This tutorial uses AWS free tier resources as much as possible - see the [cost considerations](README.md#cost-considerations) in the README for full details.

### Step 1: Clone and Prepare the Application

```bash
# If you haven't cloned the repository yet:
git clone https://github.com/worshamweb/aws-deploying-a-rails-app-to-ecs-with-copilot-tutorial
cd aws-deploying-a-rails-app-to-ecs-with-copilot-tutorial

# If you've already cloned it, just navigate to the directory:
# cd aws-deploying-a-rails-app-to-ecs-with-copilot-tutorial

# Verify Docker is working
docker --version
# Expected output: Docker version 20.x.x or higher

# Test the application locally (optional but recommended)
docker build -t weather-app .
# This will take 2-3 minutes and show build progress

docker run -p 3000:80 weather-app
# Visit http://localhost:3000/forecast to test
# Press Ctrl+C to stop the container when done testing
# Note: The -p 3000:80 flag maps your computer's port 3000 to the container's port 80
# This means: localhost:3000 (your computer) → port 80 (inside container)
```

**What's happening**: We're downloading the code and testing that Docker can build and run our application locally before deploying to AWS.

💡 **FYI**: Rails 8 includes a production-ready Dockerfile by default when you create a new Rails application. If you're deploying an older Rails app without a Dockerfile, you'll need to create one, but that's outside the scope of this tutorial.

### Step 2: Create ECR Repository

```bash
# Create a private container registry for your application
# Replace YOUR_AWS_REGION with your actual AWS region (e.g., us-east-1)
aws ecr create-repository --repository-name rails-weather-app-ecs-tutorial --region YOUR_AWS_REGION
```

**Expected output**:
```json
{
    "repository": {
        "repositoryArn": "arn:aws:ecr:YOUR_AWS_REGION:YOUR_AWS_ACCOUNT_ID:repository/rails-weather-app-ecs-tutorial",
        "registryId": "YOUR_AWS_ACCOUNT_ID",
        "repositoryName": "rails-weather-app-ecs-tutorial",
        "repositoryUri": "YOUR_AWS_ACCOUNT_ID.dkr.ecr.YOUR_AWS_REGION.amazonaws.com/rails-weather-app-ecs-tutorial"
    }
}
```

**What's happening**: We're creating a private container registry in Amazon ECR (Elastic Container Registry) where AWS will store your application's Docker images. This is more secure and better integrated with AWS services than using public registries like Docker Hub.

💰 **Cost Note**: Creating ECR repositories is free, and storing small Docker images (like our Rails app) falls within the 500 MB free tier. You don't need to worry about deleting this repository immediately after the tutorial. However, you might want to delete it before creating another repository in the future to stay within the free tier storage limit.

### Step 3: Initialize AWS Copilot

```bash
# Initialize Copilot in your project directory
copilot app init weather-app
```

**Expected output**:
```
✔ Proposing infrastructure changes for stack weather-app-infrastructure-roles
- Creating the infrastructure for stack weather-app-infrastructure-roles                        [create complete]  [49.1s]
  - A StackSet admin role assumed by CloudFormation to manage regional stacks                   [create complete]  [17.7s]
  - An IAM role assumed by the admin role to create ECR repositories, KMS keys, and S3 buckets  [create complete]  [24.4s]
✔ The directory copilot will hold service manifests for application weather-app.

Recommended follow-up action:
  - Run `copilot init` to add a new service or job to your application.
```

**What's happening**: Copilot creates a new "application" in AWS. Think of this as a container for all the resources your app needs. You'll see a new `copilot/` directory created with configuration files.

### Step 4: Create a Load Balanced Web Service

```bash
# Create a load-balanced web service
copilot svc init --name web --svc-type "Load Balanced Web Service"
```

**What you'll see**: Copilot will prompt you to choose a Dockerfile:
```
Note: It's best to run this command in the root of your workspace.

  Which Dockerfile would you like to use for web?  [Use arrows to move, type to filter, ? for more help]
  > ./Dockerfile
    Enter custom path for your Dockerfile
    Use an existing image instead
```

**What to do**: Select the first option `./Dockerfile` (it should be highlighted by default) and press Enter.

**Expected output**:
```
✔ Wrote the manifest for service web at copilot/web/manifest.yml
Your manifest contains configurations like your container size and port.

- Update regional resources with stack set "weather-app-infrastructure"  [succeeded]  [0.0s]
Recommended follow-up actions:
  - Update your manifest copilot/web/manifest.yml to change the defaults.
  - Run `copilot svc deploy --name web --env test` to deploy your service to a test environment.
```

**What's happening**: This creates a "service" configuration. A service is a running application that can handle web traffic. The "Load Balanced" type means AWS will automatically distribute incoming requests across multiple copies of your app for reliability.

This command creates:
- `copilot/web/manifest.yml` - Configuration file for your service
- Load balancer setup for handling web traffic
- Health check configuration to monitor your app

### Step 5: Configure the Service

The generated `copilot/web/manifest.yml` will need some adjustments for our Rails application. Open the file and update it with these changes:

```yaml
# copilot/web/manifest.yml
name: web
type: Load Balanced Web Service

# Distribute traffic to your service
http:
  path: '/'
  # Rails health check endpoint (change from default "/" to "/up")
  healthcheck: '/up'

# Configuration for your containers and service
image:
  build: Dockerfile
  port: 80  # Rails app runs on port 80 inside container

# Resource allocation
cpu: 256       # Number of CPU units for the task
memory: 512    # Amount of memory in MiB used by the task
count: 1       # Number of tasks that should be running
exec: true     # Enable running commands in your container

network:
  connect: true # Enable Service Connect for intra-environment traffic

# Environment variables for Rails
variables:
  RAILS_ENV: production
  RAILS_LOG_TO_STDOUT: true
  RAILS_SERVE_STATIC_FILES: true
```

**Key changes made**:
- **Health check**: Changed from `/` to `/up` (Rails 8's built-in health check endpoint)
- **Environment variables**: Added Rails production settings
- **Port**: Explicitly set to 80 to match our Dockerfile configuration

⚠️ **Cost Warning**: This configuration uses AWS Fargate, which is **not covered by the free tier**. With these settings (0.25 vCPU, 512 MiB memory), expect costs of approximately **$0.012/hour or ~$9/month** if running continuously. For learning purposes, consider:
- Running the tutorial and then cleaning up resources immediately
- Only running during testing periods
- See [COST_OPTIMIZATION.md](COST_OPTIMIZATION.md) for cost management strategies

### Step 6: Create an Environment

```bash
# Create a test environment
copilot env init --name test
```

**What you'll see**: Copilot will prompt you to choose credentials:
```
  Which credentials would you like to use to create test?  [Use arrows to move, type to filter, ? for more help]
  > Enter temporary credentials
    [profile default]
```

**What to do**: Use the down arrow key to select `[profile default]` (the second option) since you've already configured your AWS CLI credentials, then press Enter.

**Next, you'll see**: Copilot will ask about environment configuration:
```
 Would you like to use the default configuration for a new environment?
    - A new VPC with 2 AZs, 2 public subnets and 2 private subnets
    - A new ECS Cluster
    - New IAM Roles to manage services and jobs in your environment
  [Use arrows to move, type to filter]
  > Yes, use default.
    Yes, but I'd like configure the default resources (CIDR ranges, AZs).
    No, I'd like to import existing resources (VPC, subnets).
```

**What to do**: Select `Yes, use default.` (the first option, already highlighted) and press Enter. This creates a complete, isolated environment for your application.

💡 **Want to learn more?** If you're curious about VPC networking, CIDR ranges, and subnets, check out the [AWS VPC User Guide](https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html) to understand how AWS networking works.

**Expected output**:
```
✔ Wrote the manifest for environment test at copilot/environments/test/manifest.yml
- Update regional resources with stack set "weather-app-infrastructure"  [succeeded]  [0.0s]
- Update regional resources with stack set "weather-app-infrastructure"  [succeeded]           [41.3s]
  - Update resources in region "us-east-1"                               [create complete]     [40.4s]
    - ECR container image repository for "web"                           [create complete]     [2.1s]
    - KMS key to encrypt pipeline artifacts between stages               [create complete]     [16.8s]
    - S3 Bucket to store local artifacts                                 [create in progress]  [4.6s]
✔ Proposing infrastructure changes for the weather-app-test environment.
- Creating the infrastructure for the weather-app-test environment.  [create complete]  [42.5s]
  - An IAM Role for AWS CloudFormation to manage resources           [create complete]  [20.9s]
  - An IAM Role to describe resources in your environment            [create complete]  [19.4s]
✔ Provisioned bootstrap resources for environment test in region us-east-1 under application weather-app.
Recommended follow-up actions:
  - Update your manifest copilot/environments/test/manifest.yml to change the defaults.
  - Run `copilot env deploy --name test` to deploy your environment.
```

```bash
# Deploy the environment (this creates VPC, subnets, load balancer, etc.)
copilot env deploy --name test
```

**What's happening**: An "environment" is like a complete copy of your infrastructure (networking, security, load balancers). You might have separate environments for testing, staging, and production.

**⏱️ Time expectation**: This step takes ~5 minutes because AWS is creating:
- Virtual Private Cloud (VPC) - your private network
- Subnets - network segments for security
- Internet Gateway - connection to the internet
- Security Groups - firewall rules
- Application Load Balancer - traffic distribution

**Expected output** (this will stream for several minutes):
```
✔ Proposing infrastructure changes for the weather-app-test environment.
- Creating the infrastructure for the weather-app-test environment.                    [update complete]  [67.6s]
  - An ECS cluster to group your services                                              [create complete]  [9.1s]
  - A security group to allow your containers to talk to each other                    [create complete]  [7.4s]
  - An Internet Gateway to connect to the public internet                              [create complete]  [15.8s]
  - A resource policy to allow AWS services to create log streams for your workloads.  [create complete]  [0.0s]
  - Private subnet 1 for resources with no internet access                             [create complete]  [3.0s]
  - Private subnet 2 for resources with no internet access                             [create complete]  [3.0s]
  - A custom route table that directs network traffic for the public subnets           [create complete]  [11.1s]
  - Public subnet 1 for resources that can access the internet                         [create complete]  [3.0s]
  - Public subnet 2 for resources that can access the internet                         [create complete]  [3.0s]
  - A private DNS namespace for discovering services within the environment            [create complete]  [43.9s]
  - A Virtual Private Cloud to control networking of your AWS resources                [create complete]  [12.8s]
```

### Step 7: Set Up Secrets

```bash
# Add your Rails master key as a secret
copilot secret init --name RAILS_MASTER_KEY
```

**What you'll see**: Copilot will prompt you to enter the secret value. 

1. Open `config/master.key` in your text editor
2. Copy the entire contents (it's a long string of letters and numbers)
3. Paste it when prompted and press Enter

**Expected output**:
```
✔ Successfully put secret RAILS_MASTER_KEY in environment test as /copilot/weather-app/test/secrets/RAILS_MASTER_KEY.
You can refer to these secrets from your manifest file by editing the `secrets` section.
```
secrets:                                                                                                                                                                                                                            
    RAILS_MASTER_KEY: /copilot/\${COPILOT_APPLICATION_NAME}/\${COPILOT_ENVIRONMENT_NAME}/secrets/RAILS_MASTER_KEY                                                                                                                     
```
```

**What's happening**: Rails uses this master key to decrypt sensitive configuration data. We're storing it securely in AWS Secrets Manager so your application can access it without exposing it in your code.

### Step 8: Deploy the Service

```bash
# Deploy the web service to the test environment
copilot svc deploy --name web --env test
```

⚠️ **Docker Credentials Warning**: You may see a warning about "credentials stored unencrypted in ~/.docker/config.json". This is safe to ignore for this tutorial - these are temporary ECR tokens that expire automatically. In production environments, you should configure a Docker credential helper for better security.

**⏱️ Time expectation**: 15-20 minutes (especially on first deployment)

**What's happening** (you'll see progress for each step):
1. **Building Docker image** - Copilot builds your Rails app into a container
2. **Pushing to Amazon ECR** - Uploads the container image to AWS's container registry
3. **Creating ECS service** - Sets up the service to run your containers
4. **Configuring load balancer** - Connects your app to the internet
5. **Setting up logging** - Enables CloudWatch logs for monitoring

**Expected output** (abbreviated):
```
✔ Building your container image: docker build
✔ Pushing image to ECR.
✔ Proposing infrastructure changes for service web in environment test.
- Creating the infrastructure for service web in environment test.
  - An ECS service to run and maintain your tasks in the environment cluster.
  - A target group to connect the load balancer to your service.
  - A security group to allow your containers to talk to each other.
✔ Created environment test in region YOUR_AWS_REGION.
✔ Deployed service web.
```

### Step 9: Access Your Application

```bash
# Get the application URL
copilot svc show --name web --env test
```

**Expected output**:
```
Service name: web
Service type: Load Balanced Web Service

Configurations
  Environment   Tasks     CPU    Memory
  -----------   -----     ---    ------
  test          1         0.25   0.5Gi

Routes
  Environment   URL
  -----------   ---
  test          https://weather-app-test-YOUR_AWS_ACCOUNT_ID.YOUR_AWS_REGION.elb.amazonaws.com

Variables
  Name        Environment   Value
  ----        -----------   -----
  RAILS_ENV   test          production
```

**🎉 Success!** Copy the URL from the "Routes" section and paste it into your browser. You should see your Rails weather application running!

**Testing your deployment**:
1. Visit the URL - you should see the weather app
2. Test the health check: add `/up` to the URL (e.g., `https://your-url.com/up`)
3. You should see "Rails is up and running" or similar message

## 🔧 Configuration Files Created

After running the Copilot commands, you'll have these new files:

```
copilot/
├── .gitignore
├── environments/
│   └── test/
│       ├── addons/
│       └── copilot.yml
└── web/
    ├── copilot.yml
    └── environments/
        └── test/
            └── addons/
```

## 🔍 Monitoring and Troubleshooting

### View Logs
```bash
# View service logs
copilot svc logs --name web --env test --follow

# View logs for a specific time range
copilot svc logs --name web --env test --since 1h
```

### Check Service Status
```bash
# Show service details
copilot svc show --name web --env test

# List all services
copilot svc ls
```

### Common Issues and Solutions

1. **Build Failures**
   - Ensure Docker is running
   - Check Dockerfile syntax
   - Verify all required files are present

2. **Health Check Failures**
   - Confirm `/up` endpoint is accessible
   - Check application logs for startup errors
   - Verify RAILS_MASTER_KEY is set correctly

3. **Service Won't Start**
   - Check ECS service events in AWS Console
   - Review CloudWatch logs
   - Verify resource limits (CPU/memory)

## 🧹 Cleanup Resources

To avoid ongoing charges, clean up resources when done:

```bash
# Delete the service
copilot svc delete --name web --env test

# Delete the environment
copilot env delete --name test

# Delete the application (removes all resources)
copilot app delete weather-app
```

## 📚 Additional Resources

- [AWS Copilot Documentation](https://aws.github.io/copilot-cli/)
- [Amazon ECS Documentation](https://docs.aws.amazon.com/ecs/)
- [Rails on Docker Best Practices](https://guides.rubyonrails.org/getting_started_with_devcontainer.html)
- [AWS Free Tier Details](https://aws.amazon.com/free/)

## 🎯 Next Steps

Once your application is deployed, consider:

1. **Setting up a custom domain** with Route 53
2. **Implementing CI/CD** with GitHub Actions or AWS CodePipeline
3. **Adding a database** with Amazon RDS
4. **Implementing caching** with Amazon ElastiCache
5. **Setting up monitoring** with CloudWatch dashboards

## 🤝 Contributing

If you find issues with this tutorial or have suggestions for improvements, please open an issue or submit a pull request.

---

**Happy Deploying!** 🚀