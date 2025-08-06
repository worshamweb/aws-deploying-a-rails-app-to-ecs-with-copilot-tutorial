# AWS Copilot Quick Reference

Essential commands for managing your Rails application deployment with AWS Copilot.

## 🚀 Initial Setup Commands

```bash
# Initialize a new application
copilot app init weather-app

# Create a load-balanced web service
copilot svc init --name web --svc-type "Load Balanced Web Service"

# Initialize an environment
copilot env init --name test

# Deploy the environment (creates infrastructure)
copilot env deploy --name test
```

## 🔐 Secrets Management

```bash
# Create a new secret
copilot secret init --name RAILS_MASTER_KEY

# List secrets for an environment
copilot secret ls --env test

# Update a secret
copilot secret init --name RAILS_MASTER_KEY --overwrite
```

## 📦 Service Deployment

```bash
# Deploy service to environment
copilot svc deploy --name web --env test

# Deploy with specific image tag
copilot svc deploy --name web --env test --tag v1.2.3

# Deploy to all environments
copilot svc deploy --name web
```

## 📊 Monitoring and Debugging

```bash
# Show service details and URL
copilot svc show --name web --env test

# View real-time logs
copilot svc logs --name web --env test --follow

# View logs for specific time range
copilot svc logs --name web --env test --since 1h
copilot svc logs --name web --env test --start-time 2024-01-01T10:00:00Z

# Execute commands in running container
copilot svc exec --name web --env test --command "/bin/bash"
copilot svc exec --name web --env test --command "rails console"
```

## 📋 Information Commands

```bash
# List all applications
copilot app ls

# List all environments
copilot env ls

# List all services
copilot svc ls

# Show environment details
copilot env show --name test

# Show application overview
copilot app show weather-app
```

## 🔄 Updates and Scaling

```bash
# Update service configuration
# Edit copilot/web/copilot.yml, then:
copilot svc deploy --name web --env test

# Scale service manually (temporary)
aws ecs update-service --cluster weather-app-test --service weather-app-test-web --desired-count 3
```

## 🧪 Testing and Development

```bash
# Run a one-off task
copilot task run --image weather-app --env test

# Run task with custom command
copilot task run --image weather-app --env test --command "rails db:migrate"

# Run task with environment variables
copilot task run --image weather-app --env test --env-vars LOG_LEVEL=debug
```

## 🏗️ Pipeline Commands (CI/CD)

```bash
# Initialize a pipeline
copilot pipeline init

# Deploy the pipeline
copilot pipeline deploy

# Show pipeline status
copilot pipeline show

# Update pipeline
copilot pipeline update
```

## 🧹 Cleanup Commands

```bash
# Delete a service
copilot svc delete --name web --env test

# Delete an environment
copilot env delete --name test

# Delete entire application (removes all resources)
copilot app delete weather-app

# Force delete (skip confirmations)
copilot app delete weather-app --yes
```

## 🔧 Configuration Commands

```bash
# Generate service manifest
copilot svc init --name web --svc-type "Load Balanced Web Service" --port 80

# Package service (create CloudFormation template)
copilot svc package --name web --env test --upload-assets

# Show what resources will be created
copilot svc package --name web --env test --diff
```

## 📈 Advanced Commands

```bash
# Deploy with resource tags
copilot env deploy --name test --resource-tags department=engineering,project=weather-app

# Deploy with custom VPC
copilot env init --name prod --import-vpc-id vpc-12345 --import-private-subnet-ids subnet-12345,subnet-67890

# Use custom domain
copilot env init --name prod --import-cert-arns arn:aws:acm:region:account:certificate/cert-id
```

## 🐛 Troubleshooting Commands

```bash
# Check service health
aws ecs describe-services --cluster weather-app-test --services weather-app-test-web

# View CloudFormation events
aws cloudformation describe-stack-events --stack-name weather-app-test

# Check task definition
aws ecs describe-task-definition --task-definition weather-app-test-web

# View load balancer target health
aws elbv2 describe-target-health --target-group-arn <target-group-arn>
```

## 📝 Configuration File Locations

```
copilot/
├── .gitignore
├── environments/
│   └── test/
│       ├── copilot.yml          # Environment configuration
│       └── addons/              # Additional AWS resources
└── web/
    ├── copilot.yml              # Service configuration
    └── environments/
        └── test/
            └── addons/          # Environment-specific resources
```

## 🔍 Useful Flags

```bash
# Common flags across commands
--yes                    # Skip confirmation prompts
--follow                 # Follow logs in real-time
--since 1h              # Show logs from last hour
--env test              # Specify environment
--name web              # Specify service name
--resource-tags key=val # Add resource tags
--diff                  # Show differences before applying
```

## 💡 Pro Tips

1. **Use `--follow` with logs** for real-time debugging
2. **Always specify environment** with `--env` to avoid mistakes
3. **Use `copilot svc show`** to get your application URL quickly
4. **Check `copilot svc package --diff`** before deploying changes
5. **Use resource tags** for cost tracking and organization
6. **Keep your copilot.yml files in version control**
7. **Test locally with Docker** before deploying to AWS

## 🆘 Emergency Commands

```bash
# Stop all tasks immediately
aws ecs update-service --cluster weather-app-test --service weather-app-test-web --desired-count 0

# Rollback to previous task definition
aws ecs update-service --cluster weather-app-test --service weather-app-test-web --task-definition weather-app-test-web:1

# Delete stuck CloudFormation stack
aws cloudformation delete-stack --stack-name weather-app-test-web
```

Remember to replace `weather-app`, `web`, and `test` with your actual application, service, and environment names!