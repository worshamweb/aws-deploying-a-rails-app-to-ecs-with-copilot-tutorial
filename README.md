# AWS Copilot Tutorial: Deploying Serverless Containers to ECS Fargate

## 🌤️ Application Overview

This tutorial uses a pre-existing Rails 8.0 weather application (originally from [github.com/worshamweb/weather](https://github.com/worshamweb/weather)) to demonstrate serverless container deployment using AWS ECS Fargate with Copilot. 

**Focus**: This tutorial is about **deploying an existing Rails application**, not building one. For details about the weather app's features and functionality, see the [original repository's README](https://github.com/worshamweb/weather).

The app uses Google Places autocomplete and WeatherAPI.com to provide 7-day weather forecasts with intelligent caching.

## 🎯 What You'll Build

By following this tutorial, you'll deploy a fully functional web application to AWS that includes:
- A **containerized Rails application** running on Amazon ECS
- **Automatic load balancing** to handle traffic
- **Auto-scaling** that adjusts to demand
- **Health monitoring** and centralized logging
- **Production-ready infrastructure** using AWS best practices
- **Cost-optimized setup** using AWS free tier resources

**Estimated Time**: 45-60 minutes for first-time deployment

### About the Sample Application

For complete details about the weather application's features, architecture, and local development setup, please refer to the [original repository](https://github.com/worshamweb/weather).

**Key point**: You don't need to understand the Rails application code to complete this deployment tutorial - we'll focus entirely on the AWS deployment process.

## ✅ Prerequisites

### Required Software and Accounts

**Don't worry if you're new to these tools - we'll guide you through everything!**

**AWS Account Requirements:**
- **AWS Account with Free Tier access** - [Sign up here](https://aws.amazon.com/free/) if you don't have one
- **AWS CLI v2** - Command-line tool to interact with AWS services
  - [Installation Guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
  - Used to configure your AWS credentials and manage resources
- **AWS Copilot CLI** - Simplified tool for deploying containerized applications
  - [Installation Guide](https://aws.github.io/copilot-cli/docs/getting-started/install/)
  - Handles the complex AWS setup automatically

**Local Development Tools:**
- **Ruby 3.4.2** - [Installation guide](https://www.ruby-lang.org/en/documentation/installation/) (for local Rails app verification)
- **SQLite3** - Usually included with Ruby installations, or [install separately](https://sqlite.org/download.html)
- **Docker** - Platform for running applications in containers
  - [Install Docker Desktop](https://docs.docker.com/get-docker/) for your operating system
  - Packages your Rails app so it runs consistently anywhere
- **Git** - Version control system (you probably already have this)
  - Used for cloning the repository

### Verify Your Local Dependencies

Before starting the deployment, let's verify everything is properly installed and configured. **This mental checklist will serve you well in real-world deployments!**

#### 1. Configure AWS CLI

**Configure AWS CLI** with your credentials (unless you have done this before):
```bash
aws configure
# You'll need:
# - AWS Access Key ID (from AWS Console > Security Credentials)
# - AWS Secret Access Key (from AWS Console > Security Credentials) 
# - Default region (recommend: us-east-1 for free tier)
# - Default output format (recommend: json)
```

**Required AWS Permissions**: Your AWS user needs these permissions:
- ECS (Elastic Container Service) - Full access
- ECR (Elastic Container Registry) - Full access
- CloudFormation - Full access
- IAM - Limited access for role creation
- VPC - Full access
- Application Load Balancer - Full access

> **Tip**: If you're the AWS account owner, you already have these permissions. If not, ask your AWS administrator to grant these permissions.

#### 2. Verify Docker Installation
```bash
# Check Docker version
docker --version
# Should show: Docker version 20.x.x or higher

# Test Docker is working
docker run hello-world
# Should download and run a test container successfully
```

#### 3. Verify AWS CLI Configuration
```bash
# Check AWS CLI version
aws --version
# Should show: aws-cli/2.x.x or higher

# Check your configured region
aws configure get region
# Should show your AWS region (e.g., us-east-1)

# Verify AWS credentials are working
aws sts get-caller-identity --query Account --output text
# Should show your AWS account ID (12-digit number)
```

**⚠️ Important**: Note your AWS region and account ID - you'll reference these during deployment as YOUR_AWS_REGION and YOUR_AWS_ACCOUNT_ID!

#### 4. Verify Ruby Installation
```bash
# Check Ruby version
ruby --version
# Should show: ruby 3.4.2 or compatible version

# Check if bundler is available
bundle --version
# Should show bundler version information
```

#### 5. Verify Copilot CLI
```bash
# Check Copilot version
copilot --version
# Should show version information
```

### Verify the Local Rails App Works

```bash
# Clone the repository
# Replace <repository-url> with your actual GitHub repository URL
git clone <repository-url>
cd aws-deploying-a-rails-app-to-ecs-with-copilot-tutorial

# Install dependencies and test locally
bundle install
rails db:prepare
rails server
# Visit http://localhost:3000/forecast/ to confirm the Rails app is running as expected
```

> **Having issues with the Rails app?** Check the [original repository's README](https://github.com/worshamweb/weather) for troubleshooting local development issues.

## 🚀 Ready to Deploy?

Once all prerequisites are verified, proceed to the deployment tutorial:

**Next Step**: See [DEPLOYMENT_TUTORIAL.md](DEPLOYMENT_TUTORIAL.md) for detailed deployment instructions.

> **New to AWS?** Don't worry! The deployment tutorial is designed for beginners and explains each step in detail.

## 📚 Documentation

- **[DEPLOYMENT_TUTORIAL.md](DEPLOYMENT_TUTORIAL.md)** - Complete guide to deploying this app to AWS ECS using Copilot
- **[TROUBLESHOOTING.md](TROUBLESHOOTING.md)** - Common issues and solutions
- **[COST_OPTIMIZATION.md](COST_OPTIMIZATION.md)** - Tips to minimize AWS costs

## 🔧 Local Troubleshooting

### Cache Configuration
App uses memory cache by default. To enable SSD caching:
```bash
# Change in config/environments/development.rb:
config.cache_store = :solid_cache_store

# Create cache database
./bin/rails db:setup:cache
```

## ☁️ AWS Fargate Deployment Features

- **Serverless containers** using AWS Fargate (no server management required)
- **Container-based deployment** using Docker and Amazon ECS
- **Load balancing** with Application Load Balancer
- **Auto-scaling** based on CPU and memory usage
- **Health monitoring** with built-in health checks
- **Centralized logging** with CloudWatch
- **Secret management** for API keys and credentials
- **Infrastructure as Code** with AWS Copilot (simplified CloudFormation)

## 💰 Cost Considerations

### Free Tier Resources Used
- **Amazon ECS**: 750 hours per month ℹ️ *~31 days* of t2.micro/t3.micro instances (first 12 months)
- **Application Load Balancer**: 750 hours per month ℹ️ *~31 days* (first 12 months)
- **Amazon ECR**: 500 MB of storage per month (always free)
- **CloudWatch Logs**: First 5 GB per month free (always free)
- **Data Transfer**: Minimal for testing, but monitor usage

### Fargate Costs (Not Free Tier)
- **ECS Fargate**: ~$0.04048 per vCPU per hour, ~$0.004445 per GB memory per hour
- **Tutorial configuration**: 0.25 vCPU + 512MB = ~$0.012/hour (~$9/month if left running)
- **Application Load Balancer**: 750 hours per month free (first 12 months)
- **Amazon ECR**: 500 MB storage per month free
- **CloudWatch Logs**: First 5 GB per month free
- **Data Transfer**: Minimal for testing

> **⚠️ Important**: This tutorial uses AWS Fargate, which is **not covered by the free tier**. However, costs are predictable and low (~$9/month for the small container we deploy).
>
> **For learning**: Complete the tutorial, test your deployment, then clean up resources immediately to minimize costs. We'll show you exactly how to delete everything at the end.

For detailed cost management strategies, see [COST_OPTIMIZATION.md](COST_OPTIMIZATION.md).

## 🎯 Learning Objectives

By completing this tutorial, you'll learn:
- How to containerize a Rails application
- AWS ECS and Fargate fundamentals
- Infrastructure as Code with AWS Copilot
- Load balancing and auto-scaling concepts
- AWS security best practices
- Cost optimization strategies
- Monitoring and troubleshooting techniques


