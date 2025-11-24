# AWS Deployment Guide - ML Project with Elastic Beanstalk & CodePipeline

This guide will walk you through deploying your ML project to AWS using Elastic Beanstalk and CodePipeline for continuous deployment.

## Prerequisites

Before starting, ensure you have:
1. An AWS account (create one at https://aws.amazon.com if you don't have one)
2. Git installed on your computer
3. AWS CLI installed (optional but helpful)
4. Your model artifacts (preprocessor.pkl, model.pkl) ready

## Overview

- **Elastic Beanstalk**: AWS service that handles deployment, scaling, and management of your Flask application
- **CodePipeline**: AWS service for continuous integration/continuous deployment (CI/CD) that automatically deploys your code when you push changes

---

## Part 1: Initial Setup (One-Time Configuration)

### Step 1: Create a GitHub Repository (or use another Git provider)

1. Go to GitHub (https://github.com) and sign in
2. Click the "+" icon in the top right → "New repository"
3. Name your repository (e.g., "mlproject")
4. Choose "Public" or "Private"
5. Click "Create repository"
6. **Do NOT initialize with README** (since you already have code)

### Step 2: Push Your Code to GitHub

Open PowerShell in your project directory and run:

```powershell
# Initialize git repository if not already done
git init

# Add all files
git add .

# Commit your changes
git commit -m "Initial commit for AWS deployment"

# Add your GitHub repository as remote (replace with your repository URL)
git remote add origin https://github.com/YOUR_USERNAME/mlproject.git

# Push to GitHub
git branch -M main
git push -u origin main
```

**Important**: Make sure your trained model files (`preprocessor.pkl`, `model.pkl`) are in the `artifacts` folder before pushing.

---

## Part 2: AWS Account Setup

### Step 3: Sign in to AWS Console

1. Go to https://console.aws.amazon.com
2. Sign in with your AWS account
3. Select a region (e.g., US East N. Virginia - us-east-1)
   - **Keep the same region throughout this guide**

### Step 4: Create an IAM User (Security Best Practice)

1. In AWS Console, search for "IAM" in the top search bar
2. Click "Users" in the left sidebar → "Create user"
3. Username: `elasticbeanstalk-user`
4. Check "Provide user access to the AWS Management Console"
5. Select "I want to create an IAM user"
6. Click "Next"
7. Select "Attach policies directly"
8. Search and select these policies:
   - `AWSElasticBeanstalkFullAccess`
   - `AWSCodePipelineFullAccess`
   - `AmazonS3FullAccess`
   - `AWSCodeBuildAdminAccess`
9. Click "Next" → "Create user"
10. **Save the console sign-in URL, username, and password** in a safe place

---

## Part 3: Deploy with Elastic Beanstalk

### Step 5: Create Elastic Beanstalk Application

1. In AWS Console, search for "Elastic Beanstalk"
2. Click "Create application"

**Application details:**
- Application name: `mlproject-app`
- Application tags: (optional) Key=`Project`, Value=`MLProject`

**Environment:**
- Click "Configure more options" instead of using the wizard

**Presets:**
- Select "Single instance (free tier eligible)" if you want to minimize costs
- Or select "High availability" for production

**Platform:**
- Platform: `Python`
- Platform branch: `Python 3.9 running on 64bit Amazon Linux 2`
- Platform version: (use recommended/latest)

3. Click "Create application" (This takes 5-10 minutes)

### Step 6: Upload Your Application

Once your environment is created:

1. **Prepare deployment package:**
   - Open PowerShell in your project directory
   - Create a zip file with all necessary files:
   
   ```powershell
   # Create a zip file excluding unnecessary folders
   Compress-Archive -Path application.py,requirements.txt,setup.py,src,templates,.ebextensions,artifacts -DestinationPath deploy.zip -Force
   ```

2. **Upload to Elastic Beanstalk:**
   - In the Elastic Beanstalk dashboard, click your environment name
   - Click "Upload and deploy"
   - Choose file: Select `deploy.zip`
   - Version label: `v1.0`
   - Click "Deploy"
   - Wait 3-5 minutes for deployment

3. **Test your application:**
   - Once deployment is complete, you'll see a URL like: `http://mlproject-app.us-east-1.elasticbeanstalk.com`
   - Click the URL to test your application
   - Test the `/predict` endpoint

---

## Part 4: Set Up CodePipeline for Continuous Deployment

### Step 7: Create CodePipeline

1. In AWS Console, search for "CodePipeline"
2. Click "Create pipeline"

**Pipeline settings:**
- Pipeline name: `mlproject-pipeline`
- Service role: "New service role" (auto-populated)
- Allow AWS CodePipeline to create a service role: ✓
- Advanced settings: (leave default)
- Click "Next"

### Step 8: Add Source Stage

**Source provider:** `GitHub (Version 2)` (recommended)
   
1. Click "Connect to GitHub"
2. Connection name: `github-connection`
3. Click "Connect to GitHub"
4. Authorize AWS Connector
5. Click "Install a new app"
6. Select your GitHub account
7. Choose "Only select repositories" → select your `mlproject` repository
8. Click "Save"
9. Back in AWS, click "Connect"

**After connection is established:**
- Repository name: Select your repository (e.g., `YOUR_USERNAME/mlproject`)
- Branch name: `main`
- Change detection options: "Start the pipeline on source code change" (checked)
- Output artifact format: "CodePipeline default"
- Click "Next"

### Step 9: Add Build Stage

**Build provider:** `AWS CodeBuild`

1. Click "Create project"
2. Project name: `mlproject-build`
3. Environment:
   - Environment image: "Managed image"
   - Operating system: "Amazon Linux 2"
   - Runtime: "Standard"
   - Image: "aws/codebuild/amazonlinux2-x86_64-standard:4.0" (or latest)
   - Image version: "Always use the latest image"
   - Service role: "New service role"
4. Buildspec:
   - Build specifications: "Use a buildspec file"
   - Buildspec name: `buildspec.yml` (already in your repo)
5. Click "Continue to CodePipeline"
6. Click "Next"

### Step 10: Add Deploy Stage

**Deploy provider:** `AWS Elastic Beanstalk`

- Application name: `mlproject-app` (select from dropdown)
- Environment name: Select your environment (e.g., `Mlproject-app-env`)
- Click "Next"

### Step 11: Review and Create

1. Review all settings
2. Click "Create pipeline"
3. The pipeline will automatically start running

**Pipeline stages:**
1. **Source**: Pulls code from GitHub
2. **Build**: Runs tests and packages application (using buildspec.yml)
3. **Deploy**: Deploys to Elastic Beanstalk

Wait for all stages to complete (green checkmarks). This may take 5-10 minutes.

---

## Part 5: Testing and Ongoing Usage

### Step 12: Test Your Deployed Application

1. Get your application URL:
   - Go to Elastic Beanstalk dashboard
   - Click your environment
   - Copy the URL (e.g., `http://mlproject-app.us-east-1.elasticbeanstalk.com`)

2. Test in browser:
   - Homepage: `http://your-url.elasticbeanstalk.com/`
   - Prediction page: `http://your-url.elasticbeanstalk.com/predict`

3. Test prediction functionality with sample data

### Step 13: Make Changes and Auto-Deploy

Now whenever you make changes:

```powershell
# Make your code changes

# Add and commit
git add .
git commit -m "Updated model/feature/etc"

# Push to GitHub
git push origin main
```

**CodePipeline will automatically:**
1. Detect the change in GitHub
2. Pull the new code
3. Build and test
4. Deploy to Elastic Beanstalk

You can watch the progress in the CodePipeline console.

---

## Part 6: Monitoring and Troubleshooting

### View Application Logs

1. Go to Elastic Beanstalk dashboard
2. Click your environment
3. Click "Logs" in left sidebar
4. Click "Request Logs" → "Last 100 Lines" or "Full Logs"
5. Download and review logs for errors

### View Pipeline Execution

1. Go to CodePipeline dashboard
2. Click your pipeline name
3. Click any stage to see details
4. Click "View in CodeBuild" to see build logs

### Common Issues and Solutions

**Issue: Application shows 502 Bad Gateway**
- Check logs for Python errors
- Ensure `application.py` has `application = Flask(__name__)` and `app = application`
- Verify all dependencies are in requirements.txt

**Issue: Model not found error**
- Ensure `artifacts` folder with model files is included in deployment
- Check that artifacts folder is NOT in .gitignore
- Verify model paths in your code

**Issue: Pipeline fails at Build stage**
- Check buildspec.yml syntax
- Review CodeBuild logs for specific errors
- Ensure Python version matches your development environment

**Issue: Memory or timeout errors**
- In Elastic Beanstalk, go to Configuration → Instances
- Increase instance type (t2.small or t2.medium)
- In Configuration → Software, increase timeout

---

## Part 7: Cost Management

### Free Tier Usage
- Elastic Beanstalk itself is free
- You pay for underlying resources (EC2, S3)
- AWS Free Tier includes:
  - 750 hours/month of t2.micro EC2 instances (1 year)
  - 5GB of S3 storage
  - 1 million CodeBuild minutes (free tier)

### To Minimize Costs:
1. Use "Single instance" environment (not load balanced)
2. Use t2.micro instance type
3. Delete unused environments
4. Stop/terminate environment when not in use:
   - Go to Elastic Beanstalk → Actions → "Terminate environment"

### Monitor Costs:
- AWS Console → Billing Dashboard
- Set up billing alerts for spending over $10, $50, etc.

---

## Part 8: Best Practices

### Security
- Never commit AWS credentials to Git
- Use IAM roles for AWS services
- Enable HTTPS in production (use AWS Certificate Manager)
- Restrict security groups to necessary ports

### Version Control
- Use semantic versioning (v1.0.0, v1.1.0, v2.0.0)
- Tag releases in Git: `git tag v1.0.0` and `git push --tags`
- Keep a CHANGELOG.md

### Monitoring
- Set up CloudWatch alarms for errors
- Monitor application health in Elastic Beanstalk dashboard
- Set up SNS notifications for pipeline failures

### Scaling
- Use environment configuration to set up auto-scaling rules
- Monitor CPU and memory usage
- Consider load balancing for production traffic

---

## Quick Reference Commands

```powershell
# Push changes to trigger deployment
git add .
git commit -m "Your message"
git push origin main

# Create deployment package manually
Compress-Archive -Path application.py,requirements.txt,setup.py,src,templates,.ebextensions,artifacts -DestinationPath deploy.zip -Force

# Check git status
git status

# View commit history
git log --oneline
```

---

## Useful AWS Console Links

- Elastic Beanstalk: https://console.aws.amazon.com/elasticbeanstalk
- CodePipeline: https://console.aws.amazon.com/codesuite/codepipeline/pipelines
- CloudWatch Logs: https://console.aws.amazon.com/cloudwatch
- IAM: https://console.aws.amazon.com/iam
- Billing: https://console.aws.amazon.com/billing

---

## Support and Resources

- AWS Elastic Beanstalk Documentation: https://docs.aws.amazon.com/elasticbeanstalk
- AWS CodePipeline Documentation: https://docs.aws.amazon.com/codepipeline
- AWS Free Tier: https://aws.amazon.com/free
- Flask Deployment: https://flask.palletsprojects.com/en/2.3.x/deploying/

---

## Summary

You now have:
✅ Your ML application deployed on AWS Elastic Beanstalk
✅ Automated CI/CD pipeline with CodePipeline
✅ Automatic deployments when you push to GitHub
✅ A publicly accessible URL for your application

**Next Steps:**
1. Test your application thoroughly
2. Set up custom domain (optional)
3. Enable HTTPS (recommended for production)
4. Set up monitoring and alerts
5. Configure auto-scaling if needed

Good luck with your deployment! 🚀
