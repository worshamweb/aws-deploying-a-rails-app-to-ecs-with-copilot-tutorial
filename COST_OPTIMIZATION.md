# Cost Optimization Guide

This guide helps you minimize costs while deploying and running your Rails application on AWS using Copilot.

## 💰 Understanding AWS Costs

### Free Tier Resources (First 12 Months)
- **Amazon ECS**: 750 hours ℹ️ *~31 days* of t2.micro/t3.micro instances
- **Application Load Balancer**: 750 hours per month ℹ️ *~31 days*
- **Amazon ECR**: 500 MB of storage per month (always free)
- **CloudWatch Logs**: 5 GB of log ingestion per month (always free)
- **Data Transfer**: 1 GB per month (always free)

### Potential Costs After Free Tier
- **ECS Fargate**: ~$0.04048 per vCPU per hour, ~$0.004445 per GB memory per hour
- **Application Load Balancer**: ~$0.0225 per hour
- **NAT Gateway**: ~$0.045 per hour (if using private subnets)
- **Data Transfer**: $0.09 per GB (outbound)

## 🎯 Cost Optimization Strategies

### 1. Right-Size Your Resources

#### Optimize CPU and Memory
```yaml
# copilot/web/copilot.yml - Start small and scale up if needed
cpu: 256      # 0.25 vCPU (minimum)
memory: 512   # 512 MB (minimum)

# Monitor and adjust based on actual usage
count:
  min: 1      # Start with single instance
  max: 3      # Limit maximum instances
```

#### Use Spot Instances (Advanced)
```yaml
# For non-production workloads
platform:
  osfamily: linux
  architecture: x86_64
capacity_providers:
  - name: FARGATE_SPOT
    weight: 1
```

### 2. Optimize Networking Costs

#### Use Public Subnets When Possible
```yaml
# copilot/web/copilot.yml
network:
  vpc:
    placement: 'public'  # Avoids NAT Gateway costs
```

#### Minimize Data Transfer
- Use CloudFront for static assets (free tier: 1TB/month)
- Compress responses in your Rails app
- Optimize image sizes and formats

### 3. Efficient Logging Strategy

#### Reduce Log Retention
```yaml
# copilot/environments/test/copilot.yml
network:
  vpc:
    enable_logs: true
    log_retention: 7  # days (default is 30)
```

#### Filter Logs
```ruby
# config/environments/production.rb
config.log_level = :warn  # Reduce log verbosity
```

### 4. Container Optimization

#### Multi-Stage Docker Build
Your existing Dockerfile already uses multi-stage builds, which is excellent for reducing image size:

```dockerfile
# Optimizations already in place:
# - Multi-stage build reduces final image size
# - Removes build dependencies from final image
# - Uses slim base image
```

#### Further Optimizations
```dockerfile
# Add to Dockerfile for even smaller images
RUN apt-get clean && \
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# Use .dockerignore to exclude unnecessary files
```

### 5. Auto-Scaling Configuration

#### Conservative Scaling
```yaml
# copilot/web/copilot.yml
count:
  min: 1
  max: 3
  cooldown:
    scale_in_cooldown: 300s   # 5 minutes
    scale_out_cooldown: 60s   # 1 minute
  target_cpu: 80              # Scale at 80% CPU
  target_memory: 85           # Scale at 85% memory
```

### 6. Development vs Production Environments

#### Use Smaller Resources for Development
```yaml
# copilot/web/environments/dev/copilot.yml
cpu: 256
memory: 512
count: 1

# copilot/web/environments/prod/copilot.yml
cpu: 512
memory: 1024
count:
  min: 2
  max: 10
```

## 📊 Cost Monitoring

### 1. Set Up Billing Alerts

#### AWS CLI Method
```bash
# Create billing alarm for $10 threshold
aws cloudwatch put-metric-alarm \
  --alarm-name "weather-app-billing-alarm" \
  --alarm-description "Billing alarm for weather app" \
  --metric-name EstimatedCharges \
  --namespace AWS/Billing \
  --statistic Maximum \
  --period 86400 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=Currency,Value=USD \
  --alarm-actions arn:aws:sns:us-east-1:ACCOUNT-ID:billing-alarm
```

#### AWS Console Method
1. Go to CloudWatch > Alarms
2. Create Alarm > Select Metric > Billing > Total Estimated Charge
3. Set threshold (e.g., $10)
4. Configure SNS notification

### 2. Cost Analysis Commands

```bash
# Get current month costs
aws ce get-cost-and-usage \
  --time-period Start=$(date +%Y-%m-01),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --group-by Type=DIMENSION,Key=SERVICE

# Get daily costs for last 7 days
aws ce get-cost-and-usage \
  --time-period Start=$(date -d '7 days ago' +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity DAILY \
  --metrics BlendedCost
```

### 3. Resource Tagging for Cost Tracking

```yaml
# copilot/environments/test/copilot.yml
network:
  vpc:
    tags:
      Project: weather-app
      Environment: test
      Owner: your-name
      CostCenter: development
```

## 🔄 Cleanup Strategies

### 1. Automated Cleanup Script

```bash
#!/bin/bash
# scripts/cleanup.sh

echo "🧹 Cleaning up AWS resources..."

# Delete service
copilot svc delete --name web --env test --yes

# Delete environment
copilot env delete --name test --yes

# Delete application (removes all resources)
copilot app delete weather-app --yes

echo "✅ Cleanup complete!"
```

### 2. Scheduled Cleanup (Development)

```yaml
# .github/workflows/cleanup.yml
name: Nightly Cleanup
on:
  schedule:
    - cron: '0 2 * * *'  # 2 AM daily
jobs:
  cleanup:
    runs-on: ubuntu-latest
    steps:
      - name: Cleanup dev environment
        run: |
          copilot svc delete --name web --env dev --yes
```

### 3. Manual Cleanup Checklist

Before leaving your AWS account:
- [ ] Delete Copilot services: `copilot svc delete`
- [ ] Delete environments: `copilot env delete`
- [ ] Delete applications: `copilot app delete`
- [ ] Check CloudFormation stacks
- [ ] Verify ECR repositories are deleted
- [ ] Check for orphaned resources in EC2, VPC

## 💡 Cost-Saving Tips

### 1. Development Best Practices
- Use local development when possible
- Share development environments among team members
- Use smaller instance types for testing
- Delete resources when not in use

### 2. Production Optimizations
- Use Reserved Instances for predictable workloads
- Implement proper caching strategies
- Use CloudFront for static content delivery
- Monitor and optimize database queries

### 3. Alternative Architectures
For very low-cost deployments, consider:
- **AWS Lambda** with Rails API mode
- **AWS App Runner** for simpler deployments
- **Heroku** for rapid prototyping (though not AWS)

## 📈 Cost Estimation Tool

Create a simple cost calculator:

```bash
#!/bin/bash
# scripts/cost-calculator.sh

echo "💰 AWS Cost Calculator for Weather App"
echo "======================================"

read -p "Hours per month the app will run: " hours
read -p "Number of tasks (containers): " tasks
read -p "CPU units (256, 512, 1024): " cpu
read -p "Memory MB (512, 1024, 2048): " memory

# Fargate pricing (us-east-1)
cpu_cost_per_hour=0.04048
memory_cost_per_gb_hour=0.004445
alb_cost_per_hour=0.0225

# Calculate costs
vcpu=$(echo "scale=2; $cpu / 1024" | bc)
memory_gb=$(echo "scale=2; $memory / 1024" | bc)

fargate_cost=$(echo "scale=2; $hours * $tasks * ($vcpu * $cpu_cost_per_hour + $memory_gb * $memory_cost_per_gb_hour)" | bc)
alb_cost=$(echo "scale=2; $hours * $alb_cost_per_hour" | bc)
total_cost=$(echo "scale=2; $fargate_cost + $alb_cost" | bc)

echo
echo "Estimated Monthly Costs:"
echo "Fargate: \$$fargate_cost"
echo "Load Balancer: \$$alb_cost"
echo "Total: \$$total_cost"
echo
echo "Note: This excludes data transfer, CloudWatch logs, and other services"
```

## 🎯 Free Tier Maximization

### Stay Within Free Tier Limits
- Monitor usage in AWS Billing Dashboard
- Set up alerts at 80% of free tier limits
- Use AWS Free Tier Usage Alerts
- Regularly review AWS Cost Explorer

### Free Tier Tracking
```bash
# Check free tier usage
aws support describe-trusted-advisor-checks --language en --query 'checks[?name==`Service Limits`]'
```

Remember: The goal is to learn and experiment cost-effectively. Always clean up resources when you're done, and monitor your usage regularly!