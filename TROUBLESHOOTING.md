# Troubleshooting Guide

This guide covers common issues you might encounter when deploying the Rails weather application to Amazon ECS using AWS Copilot.

## 🔧 Common Issues and Solutions

### Common Beginner Mistakes

#### Forgetting to Start Docker
**Error**: `Cannot connect to the Docker daemon`
**Solution**: Make sure Docker Desktop is running (you should see the Docker icon in your system tray)

#### Wrong Directory
**Error**: `No such file or directory` when running commands
**Solution**: Make sure you're in the project directory (`aws-deploying-a-rails-app-to-ecs-with-copilot-tutorial`)

#### Skipping the Setup Script
**Problem**: Commands fail with permission or configuration errors
**Solution**: Always run `./scripts/setup.sh` first to verify your setup

#### Not Waiting for Environment Deployment
**Error**: Environment not found when deploying service
**Solution**: Wait for `copilot env deploy --name test` to complete (10-15 minutes) before proceeding

#### Copying Master Key Incorrectly
**Error**: Rails fails to start with credential errors
**Solution**: 
- Open `config/master.key` in a text editor
- Copy the ENTIRE contents (no extra spaces or newlines)
- Paste exactly what you copied

#### Using Wrong Region
**Problem**: Resources created in unexpected region
**Solution**: Check your AWS CLI configuration with `aws configure list`

### 1. Prerequisites Issues

#### AWS CLI Not Configured
**Error**: `Unable to locate credentials`
**Solution**:
```bash
aws configure
# Enter your AWS Access Key ID, Secret Access Key, region, and output format
```

#### Docker Not Running
**Error**: `Cannot connect to the Docker daemon`
**Solution**:
- Start Docker Desktop (Windows/Mac)
- Start Docker service (Linux): `sudo systemctl start docker`

#### Copilot CLI Not Found
**Error**: `copilot: command not found`
**Solution**:
```bash
# Linux/Mac
curl -Lo copilot https://github.com/aws/copilot-cli/releases/latest/download/copilot-linux
chmod +x copilot && sudo mv copilot /usr/local/bin

# Or use package managers
# Mac: brew install aws/tap/copilot-cli
# Windows: scoop install copilot
```

### 2. Application Build Issues

#### Docker Build Failures
**Error**: `failed to build image`
**Common Causes**:
- Missing Dockerfile
- Invalid Dockerfile syntax
- Network issues during gem installation

**Solutions**:
```bash
# Test Docker build locally
docker build -t weather-app .

# Check Dockerfile syntax
docker build --no-cache -t weather-app .

# For network issues, try building with different base image
```

#### Missing Log/Storage Directories
**Error**: `chown: cannot access 'log': No such file or directory`
**Cause**: Rails app missing `log` and `storage` directories
**Solution**: The Dockerfile has been updated to create these directories automatically. If you encounter this error:
```bash
# Create directories manually if needed
mkdir -p log storage

# Then rebuild
docker build -t weather-app .
```

#### Rails Bin Scripts Permission Denied
**Error**: `permission denied` for files like `/rails/bin/docker-entrypoint`, `/rails/bin/thrust`, etc.
**Cause**: Rails bin scripts lack execute permissions
**Solution**: The Dockerfile has been updated to fix this automatically. If you encounter this error:
```bash
# Fix permissions for all bin scripts
chmod +x bin/*

# Then rebuild
docker build -t weather-app .
```

#### Missing Rails Master Key
**Error**: `Rails master key not found`
**Solution**:
```bash
# Generate new credentials
rails credentials:edit

# Or copy existing master key to config/master.key
# Make sure config/master.key is in your repository
```

### 3. Copilot Deployment Issues

#### Environment Creation Fails
**Error**: `failed to create environment`
**Common Causes**:
- Insufficient AWS permissions
- Region limits exceeded
- VPC limits reached

**Solutions**:
```bash
# Check AWS permissions
aws sts get-caller-identity

# Try different region
copilot env init --name test --region us-west-2

# Check service quotas in AWS Console
```

#### Service Health Check Failures
**Error**: `service failed health checks`
**Common Causes**:
- Application not starting properly
- Wrong health check endpoint
- Port configuration issues

**Solutions**:
```bash
# Check service logs
copilot svc logs --name web --env test --follow

# Verify health check endpoint
curl http://localhost:3000/up  # Test locally first

# Check port configuration in copilot.yml
```

#### Secret Not Found
**Error**: `secret not found`
**Solution**:
```bash
# Create the secret
copilot secret init --name RAILS_MASTER_KEY

# Verify secret exists
copilot secret ls --env test
```

### 4. Runtime Issues

#### Application Won't Start
**Symptoms**: Service shows as unhealthy, logs show startup errors

**Debugging Steps**:
```bash
# Check detailed logs
copilot svc logs --name web --env test --since 1h

# Check ECS service events
aws ecs describe-services --cluster weather-app-test --services weather-app-test-web

# Test container locally
docker run -p 3000:80 -e RAILS_MASTER_KEY=<your-key> weather-app
```

#### Memory/CPU Issues
**Error**: `Task stopped due to resource constraints`
**Solution**:
Update `copilot/web/copilot.yml`:
```yaml
cpu: 512
memory: 1024
```

#### Database Connection Issues
**Error**: `database connection failed`
**Solution**:
- Ensure SQLite database is properly initialized in container
- Check file permissions for database files
- Consider using Amazon RDS for production

### 5. Load Balancer Issues

#### 502/503 Errors
**Common Causes**:
- Application not responding on correct port
- Health check failing
- Target group misconfiguration

**Solutions**:
```bash
# Verify application port
# Rails app should listen on port 80 (as configured in Dockerfile)

# Check target group health in AWS Console
# EC2 > Load Balancers > Target Groups

# Verify health check path
curl http://<load-balancer-url>/up
```

#### SSL/HTTPS Issues
**Error**: `SSL certificate not found`
**Solution**:
- Use HTTP for testing initially
- For HTTPS, configure certificate in copilot.yml:
```yaml
http:
  certificate_arn: "arn:aws:acm:region:account:certificate/cert-id"
```

### 6. Cost Management Issues

#### Unexpected Charges
**Prevention**:
```bash
# Set up billing alerts in AWS Console
# Monitor usage with:
aws ce get-cost-and-usage --time-period Start=2024-01-01,End=2024-01-31 --granularity MONTHLY --metrics BlendedCost

# Clean up resources when done
copilot svc delete --name web --env test
copilot env delete --name test
copilot app delete weather-app
```

## 🔍 Debugging Commands

### Essential Debugging Commands
```bash
# View service status
copilot svc show --name web --env test

# View real-time logs
copilot svc logs --name web --env test --follow

# List all resources
copilot env ls
copilot svc ls

# Check task status
copilot task run --image weather-app --env test

# Execute commands in running container
copilot svc exec --name web --env test --command "/bin/bash"
```

### AWS Console Debugging
1. **ECS Console**: Check service status, task definitions, and logs
2. **CloudWatch**: View detailed logs and metrics
3. **EC2 Load Balancers**: Check target group health
4. **ECR**: Verify image uploads
5. **CloudFormation**: Review stack events for deployment issues

## 📊 Monitoring and Alerts

### Set Up CloudWatch Alarms
```bash
# CPU utilization alarm
aws cloudwatch put-metric-alarm \
  --alarm-name "weather-app-high-cpu" \
  --alarm-description "High CPU utilization" \
  --metric-name CPUUtilization \
  --namespace AWS/ECS \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold
```

### Log Analysis
```bash
# Search logs for errors
copilot svc logs --name web --env test --since 1h | grep ERROR

# Filter logs by timestamp
copilot svc logs --name web --env test --start-time 2024-01-01T10:00:00Z --end-time 2024-01-01T11:00:00Z
```

## 🆘 Getting Help

### Official Resources
- [AWS Copilot Documentation](https://aws.github.io/copilot-cli/)
- [AWS ECS Documentation](https://docs.aws.amazon.com/ecs/)
- [AWS Support](https://aws.amazon.com/support/)

### Community Resources
- [AWS Copilot GitHub Issues](https://github.com/aws/copilot-cli/issues)
- [AWS Developer Forums](https://forums.aws.amazon.com/)
- [Stack Overflow](https://stackoverflow.com/questions/tagged/aws-copilot)

### Emergency Cleanup
If you need to quickly remove all resources:
```bash
# Nuclear option - removes everything
copilot app delete weather-app --yes

# Or manually delete CloudFormation stacks in AWS Console
```

## 📝 Reporting Issues

When reporting issues, include:
1. Copilot version: `copilot --version`
2. AWS CLI version: `aws --version`
3. Error messages and logs
4. Steps to reproduce
5. Your copilot.yml configuration

Remember: Most issues are related to configuration or permissions. Double-check your setup before seeking help!