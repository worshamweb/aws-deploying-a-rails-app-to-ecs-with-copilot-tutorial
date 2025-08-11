# Deployment Checklist

Use this checklist to ensure a smooth deployment of your Rails weather application to AWS ECS using Copilot.

## ✅ Pre-Deployment Checklist

### Prerequisites Verification
- [ ] AWS CLI v2 installed and configured
  - Run: `aws --version` (should show version 2.x.x)
  - Run: `aws configure` if not set up
- [ ] AWS credentials working
  - Run: `aws sts get-caller-identity` (should show your account info)
- [ ] Docker installed and running
  - Run: `docker --version` (should show version info)
  - Run: `docker info` (should not show connection errors)
- [ ] AWS Copilot CLI installed
  - Run: `copilot --version` (should show version info)
- [ ] Git installed
  - Run: `git --version` (should show version info)

### Application Preparation
- [ ] Repository cloned locally
  - You should be in the project directory
- [ ] `config/master.key` file exists
  - Check: `ls config/master.key` (file should exist)
- [ ] Dockerfile builds successfully
  - Run: `docker build -t weather-app .` (should complete without errors)
- [ ] Application runs locally in Docker
  - Run: `docker run -p 3000:80 weather-app`
  - Visit: http://localhost:3000/forecast (should load the app)
- [ ] Health check endpoint responds correctly
  - Visit: http://localhost:3000/up (should show "Rails is up")

### AWS Account Setup
- [ ] AWS account has appropriate permissions
- [ ] Billing alerts configured (recommended: $10 threshold)
- [ ] Free tier usage monitored
- [ ] Region selected (recommend: YOUR_AWS_REGION for free tier)

## 🚀 Deployment Steps

### Step 1: Initialize Copilot Application
- [ ] Run `copilot app init weather-app`
  - Expected: "✓ Created the infrastructure to manage services..."
- [ ] Verify `copilot/` directory created
  - Check: `ls copilot/` (directory should exist)
- [ ] Check `.gitignore` updated
  - The copilot directory should be tracked in git

### Step 2: Create Web Service
- [ ] Run `copilot svc init --name web --svc-type "Load Balanced Web Service"`
  - Expected: "✓ Wrote the manifest for service web..."
- [ ] Review generated `copilot/web/copilot.yml`
  - File should exist and contain service configuration
- [ ] Customize configuration if needed
  - Default settings (256 CPU, 512 MB memory) are fine for testing

### Step 3: Create Environment
- [ ] Run `copilot env init --name test`
  - Expected: "✓ Wrote the manifest for environment test..."
- [ ] Run `copilot env deploy --name test`
  - **This takes 10-15 minutes** - be patient!
  - Expected: "✓ Created environment test in region..."
- [ ] Verify environment in AWS Console (optional)
  - Go to AWS Console > CloudFormation > Stacks
  - Look for "weather-app-test" stack

### Step 4: Configure Secrets
- [ ] Run `copilot secret init --name RAILS_MASTER_KEY`
  - When prompted, paste the entire contents of `config/master.key`
  - Expected: "✓ Successfully created secret RAILS_MASTER_KEY..."
- [ ] Verify secret created
  - Run: `copilot secret ls --env test`
  - Should show RAILS_MASTER_KEY in the list

### Step 5: Deploy Service
- [ ] Run `copilot svc deploy --name web --env test`
  - **This takes 5-10 minutes** - you'll see build and deployment progress
  - Expected: "✓ Deployed service web."
- [ ] Monitor deployment progress
  - Watch for "Building your container image" and "Pushing image to ECR"
- [ ] Wait for deployment completion
  - Don't worry if it seems slow - AWS is doing a lot behind the scenes!

### Step 6: Verify Deployment
- [ ] Run `copilot svc show --name web --env test`
  - Look for the URL in the "Routes" section
- [ ] Copy the application URL
  - It will look like: https://weather-app-test-xxxxx.YOUR_AWS_REGION.elb.amazonaws.com
- [ ] Test application in browser
  - Paste the URL and visit it - you should see the weather app!
- [ ] Verify health check endpoint works
  - Add `/up` to your URL - should show "Rails is up and running"
- [ ] Check logs (optional)
  - Run: `copilot svc logs --name web --env test`
  - Should show Rails startup logs

## 🔍 Post-Deployment Verification

### Application Testing
- [ ] Application loads successfully
- [ ] Health check `/up` returns 200 OK
- [ ] Weather forecast functionality works (if API keys configured)
- [ ] No error messages in logs
- [ ] Load balancer health checks passing

### AWS Resources Verification
- [ ] ECS service running in AWS Console
- [ ] Load balancer created and healthy
- [ ] Target group shows healthy targets
- [ ] CloudWatch logs receiving data
- [ ] ECR repository contains image

### Cost Monitoring
- [ ] Check AWS billing dashboard
- [ ] Verify free tier usage
- [ ] Confirm no unexpected charges
- [ ] Set up cost alerts if not done

## 🔧 Configuration Checklist

### Security
- [ ] Secrets properly configured (not in environment variables)
- [ ] No sensitive data in logs
- [ ] HTTPS configured (if using custom domain)
- [ ] Security groups properly configured

### Performance
- [ ] Appropriate CPU/memory allocation
- [ ] Auto-scaling configured
- [ ] Health check intervals optimized
- [ ] Log retention period set

### Monitoring
- [ ] CloudWatch logs enabled
- [ ] Application metrics visible
- [ ] Alerts configured for critical issues
- [ ] Dashboard created (optional)

## 🧹 Cleanup Checklist (When Done)

### Resource Cleanup
- [ ] Run `copilot svc delete --name web --env test`
- [ ] Run `copilot env delete --name test`
- [ ] Run `copilot app delete weather-app`
- [ ] Verify CloudFormation stacks deleted
- [ ] Check for orphaned resources

### Cost Verification
- [ ] Final billing check
- [ ] Confirm all resources deleted
- [ ] No ongoing charges
- [ ] Update cost alerts if needed

## 🚨 Troubleshooting Checklist

If deployment fails, check:
- [ ] AWS credentials and permissions
- [ ] Docker daemon running
- [ ] Internet connectivity
- [ ] AWS service limits not exceeded
- [ ] Correct region selected
- [ ] No conflicting resource names

If application doesn't start:
- [ ] Check service logs
- [ ] Verify health check endpoint
- [ ] Confirm secrets are set
- [ ] Check task definition
- [ ] Verify container port configuration

If health checks fail:
- [ ] Test `/up` endpoint locally
- [ ] Check application startup time
- [ ] Verify port configuration
- [ ] Review load balancer settings
- [ ] Check security group rules

## 📋 Quick Commands Reference

```bash
# Check deployment status
copilot svc show --name web --env test

# View logs
copilot svc logs --name web --env test --follow

# Execute commands in container
copilot svc exec --name web --env test --command "/bin/bash"

# Clean up everything
copilot app delete weather-app --yes
```

## 📞 Support Resources

If you encounter issues:
- [ ] Check [TROUBLESHOOTING.md](TROUBLESHOOTING.md)
- [ ] Review [AWS Copilot Documentation](https://aws.github.io/copilot-cli/)
- [ ] Search [GitHub Issues](https://github.com/aws/copilot-cli/issues)
- [ ] Post on [AWS Forums](https://forums.aws.amazon.com/)

## 🎯 Success Criteria

Your deployment is successful when:
- [ ] Application URL loads without errors
- [ ] Health check returns 200 OK
- [ ] Logs show no critical errors
- [ ] Auto-scaling is configured
- [ ] Costs are within expected range
- [ ] All AWS resources are properly tagged

---

**Congratulations!** 🎉 You've successfully deployed a Rails application to AWS ECS using Copilot!