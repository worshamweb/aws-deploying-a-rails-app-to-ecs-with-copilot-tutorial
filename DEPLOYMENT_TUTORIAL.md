# Deploying a Rails Application to Amazon ECS with AWS Copilot

This tutorial will guide you through deploying a Rails weather application to Amazon ECS (Elastic Container Service) using AWS Copilot. The application provides weather forecasts using Google Places autocomplete and WeatherAPI.com integration.

## 🔰 First Time User? Start Here!

**Never used AWS before?** No problem! This tutorial is designed for beginners.

**What you'll need**:
- About 1 hour of time
- A computer with internet access
- An AWS account (free to create)
- Basic command line familiarity (we'll guide you through each command)

**What you'll learn**:
- How to deploy applications to the cloud
- AWS fundamentals (containers, load balancers, auto-scaling)
- Modern DevOps practices
- Cost management in the cloud

**Don't worry about**:
- Breaking anything (we use safe, isolated resources)
- Costs (tutorial designed for AWS free tier)
- Complex configurations (Copilot handles the complexity)

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
aws ecr create-repository --repository-name rails-weather-app-ecs-tutorial --region us-east-1
```

**Expected output**:
```json
{
    "repository": {
        "repositoryArn": "arn:aws:ecr:us-east-1:123456789012:repository/rails-weather-app-ecs-tutorial",
        "registryId": "123456789012",
        "repositoryName": "rails-weather-app-ecs-tutorial",
        "repositoryUri": "123456789012.dkr.ecr.us-east-1.amazonaws.com/rails-weather-app-ecs-tutorial"
    }
}
```

**What's happening**: We're creating a private container registry in Amazon ECR (Elastic Container Registry) where AWS will store your application's Docker images. This is more secure and better integrated with AWS services than using public registries like Docker Hub.

### Step 3: Initialize AWS Copilot

```bash
# Initialize Copilot in your project directory
copilot app init weather-app
```

**Expected output**:
```
✔ Created the infrastructure to manage services and jobs under application weather-app.
```

**What's happening**: Copilot creates a new "application" in AWS. Think of this as a container for all the resources your app needs. You'll see a new `copilot/` directory created with configuration files.

### Step 4: Create a Load Balanced Web Service

```bash
# Create a load-balanced web service
copilot svc init --name web --svc-type "Load Balanced Web Service"
```

**Expected output**:
```
✔ Wrote the manifest for service web at copilot/web/copilot.yml
```

**What's happening**: This creates a "service" configuration. A service is a running application that can handle web traffic. The "Load Balanced" type means AWS will automatically distribute incoming requests across multiple copies of your app for reliability.

This command creates:
- `copilot/web/copilot.yml` - Configuration file for your service
- Load balancer setup for handling web traffic
- Health check configuration to monitor your app

### Step 5: Configure the Service

The generated `copilot/web/copilot.yml` will need some adjustments for our Rails application:

```yaml
# copilot/web/copilot.yml
name: web
type: Load Balanced Web Service

http:
  healthcheck: '/up'

image:
  build: './Dockerfile'

secrets:
  - RAILS_MASTER_KEY

variables:
  RAILS_ENV: production

count:
  min: 1
  max: 10
  cooldown:
    scale_in_cooldown: 300s
    scale_out_cooldown: 300s
  target_cpu: 70
  target_memory: 80

network:
  vpc:
    enable_logs: true

exec: true
logging:
  enable_metadata: true
```

### Step 6: Create an Environment

```bash
# Create a test environment
copilot env init --name test
```

**Expected output**:
```
✔ Wrote the manifest for environment test at copilot/environments/test/copilot.yml
```

```bash
# Deploy the environment (this creates VPC, subnets, load balancer, etc.)
copilot env deploy --name test
```

**What's happening**: An "environment" is like a complete copy of your infrastructure (networking, security, load balancers). You might have separate environments for testing, staging, and production.

**⏱️ Time expectation**: This step takes 10-15 minutes because AWS is creating:
- Virtual Private Cloud (VPC) - your private network
- Subnets - network segments for security
- Internet Gateway - connection to the internet
- Security Groups - firewall rules
- Application Load Balancer - traffic distribution

**Expected output** (this will stream for several minutes):
```
✔ Proposing infrastructure changes for the weather-app-test environment.
- Creating the infrastructure for the weather-app-test environment.
  - An Internet Gateway to connect to the public internet.
  - Public subnets for internet facing services.
  - A security group to allow your containers to talk to each other.
  - A load balancer to distribute traffic to your services.
  - An ECS cluster to group your services.
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
✔ Successfully created secret RAILS_MASTER_KEY in environment test.
```

**What's happening**: Rails uses this master key to decrypt sensitive configuration data. We're storing it securely in AWS Secrets Manager so your application can access it without exposing it in your code.

### Step 8: Deploy the Service

```bash
# Deploy the web service to the test environment
copilot svc deploy --name web --env test
```

**⏱️ Time expectation**: 5-10 minutes

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
✔ Created environment test in region us-east-1.
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
  test          https://weather-app-test-123456789.us-east-1.elb.amazonaws.com

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