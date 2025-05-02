# AWS-S3-Frontend-Hosting-Guide
This step-by-step guide demonstrates how to host a React/TypeScript frontend using AWS S3 and CloudFront, including automated deployments with GitHub Actions for a scalable, secure, and cost-effective production environment.

# Hosting Your Frontend on AWS with S3 and CloudFront
## A Complete Step-by-Step Guide

This guide covers how to host your React/TypeScript frontend using Amazon S3 and CloudFront. It's scalable, cost-effective, and integrates well with AWS EC2 backend setups.

## Overview of the Approach

1. **Build Your React App**: Generate a static build of your frontend
2. **Set Up an S3 Bucket**: Store your static files in an S3 bucket configured for static website hosting
3. **Add CloudFront**: Use CloudFront as a CDN to serve your app with HTTPS and better performance
4. **Configure DNS with Route 53**: Point your custom domain to CloudFront
5. **Set Up CI/CD**: Automate the build and deployment process using GitHub Actions

## Step-by-Step Guide to Hosting Your Frontend

### Step 1: Build Your React App Locally

- Make sure there is no build error.

### Step 2: Create an S3 Bucket

1. **Log in to AWS Management Console**:
   - Go to the S3 service

2. **Create a New Bucket**:
   - Click "Create Bucket"
   - Name it something unique (e.g., `yourdomain-frontend-2025`). Bucket names must be globally unique
   - Choose your AWS region (e.g., ap-south-1 for lowest latency in the India)
   - Leave default settings for now (e.g., block public access—we'll adjust this later)
   - Click "Create Bucket"

3. **Upload the Build Files**: (skip this if you are integrating ci/cd pipeline)
   - Open your bucket in the AWS Console
   - Click "Upload," then drag and drop the entire build folder (or select all files inside it)
   - Click "Upload" to transfer the files to S3

4. **Enable Static Website Hosting**:
   - Go to the "Properties" tab of your bucket
   - Scroll to "Static website hosting" and click "Edit"
   - Enable it, set "Index document" to `index.html`, and save changes
   - Note the "Bucket website endpoint" (e.g., `http://yourdomain-frontend-2025.s3-website-us-east-1.amazonaws.com`). Test it in your browser to confirm it works

### Step 3: Make the Bucket Public

1. **Edit Block public access (bucket settings)**
   - Click on edit.
   - uncheck `Block all public access` this box.

3. **Edit Bucket Policy**:
   - Go to the "Permissions" tab of your bucket
   - Under "Bucket Policy," click "Edit" and paste this policy (replace `your-bucket-name`):

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::your-bucket-name/*"
        }
    ]
}
```

   - Save changes. This makes your static files publicly accessible

### Step 4: Set Up CloudFront

**Before going in to this stage please setup your route53 and generate the ssl certificate for you domain name from ACM**

1. **Go to CloudFront in AWS Console**:
   - Click "Create Distribution"

2. **Configure Origin**:
   - For "Origin Domain," select your S3 bucket (e.g., `yourdomain-frontend-2025.s3.amazonaws.com`)
   - Leave "Origin Access" as "Legacy access identities" for now (we'll refine this later)

3. **Default Cache Behavior**:
   - Set "Viewer Protocol Policy" to "Redirect HTTP to HTTPS" for security
   - Leave other settings as default for now

4. **Custom Domain and SSL** (Optional Now, Required Later):
   - Under "Alternate Domain Name (CNAME)," add your domain (e.g., yourdomain.com) if you have it
   - For "Custom SSL Certificate," request one via AWS Certificate Manager (ACM) in the same region as CloudFront (e.g., ap-south-1), or skip this for now and use the default CloudFront domain

5. **Default Root Object**:
   - Set to `index.html` (this ensures React Router works correctly)

6. **Create Distribution**:
   - Click "Create Distribution." It takes 5-15 minutes to deploy
   - Note the CloudFront domain (e.g., `d123456789.cloudfront.net`). Test it in your browser

### Step 6: Configure Route 53 (Custom Domain)

1. **Go to Route 53**:
   - In the AWS Console, select "Hosted Zones"
   - If you don't have a domain, register one here (e.g., yourdomain.com)

2. **Create a Record**:
   - Click "Create Record"
   - Leave "Name" blank for the root domain (e.g., yourdomain.com)
   - Set "Record Type" to A
   - Enable "Alias," then select "Alias to CloudFront distribution"
   - Choose your CloudFront distribution from the dropdown
   - Save changes

3. **Add SSL via ACM**:
   - In ACM (in us-east-1), request a certificate for yourdomain.com and *.yourdomain.com
   - Validate it via DNS (add the provided CNAME records to Route 53)
   - Once issued, edit your CloudFront distribution, add the certificate under "Custom SSL Certificate"

4. **Test**:
   - Visit https://yourdomain.com to ensure it loads securely

## Setting Up CI/CD with GitHub Actions

### Step 1: Configure AWS Credentials

1. **Create an IAM User**:
   - In AWS IAM, create a user (e.g., frontend-deployer)
   - Attach a policy with these permissions:
      - CloudFrontFullAccess
      - AmazonS3FullAccess
      - AmazonEC2FullAccess
      - AmazonEC2ContainerRegistryFullAccess

   - Generate an Access Key and Secret Key for this user

2. **Add Secrets to GitHub**:
   - In your GitHub repo, go to Settings > Secrets and variables > Actions
   - Add `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` with the values from the IAM user

### Step 2: Create a GitHub Actions Workflow

1. **Create Workflow File**:
   - In your repo, create `.github/workflows/deploy.yml` (make sure that the file name is same as you .yml file)
   - Add this content (replace placeholders):

```yaml
name: Deploy to S3 and CloudFront

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Set up Node.js
        uses: actions/setup-node@v3
        with:
          node-version: "18"

      - name: Install dependencies with legacy peer deps
        run: |
          cd client
          npm ci --legacy-peer-deps

      - name: Build
        env:
          VITE_API_BASE_URL: ${{ secrets.VITE_API_BASE_URL }}
          VITE_API_URL: ${{ secrets.VITE_API_URL }}
          // give your env credentials and add them in your github action secrets
        run: |
          cd client
          npm run build

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Deploy to S3
        run: |
          cd client
          aws s3 sync ./dist s3://${{ secrets.S3_BUCKET }} --delete

      - name: Invalidate CloudFront cache
        run: |
          aws cloudfront create-invalidation --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} --paths "/*"

```

### Step 3: Test the Pipeline

1. **Commit and Push**:
   - Push changes to your main branch
   - Go to the "Actions" tab in GitHub to monitor the workflow

2. **Verify Deployment**:
   - Once complete, visit https://yourdomain.com to confirm the update

## Industry-Standard Enhancements

1. **Error Pages**: (*make sure its added*)
   - In CloudFront, set custom error responses (e.g., 404 → /index.html, HTTP 200) to support React Router

(following are not necessary)
2. **Compression**:
   - Enable "Compress Objects Automatically" in CloudFront's behavior settings

3. **Monitoring**:
   - Use CloudFront's built-in metrics or add AWS CloudWatch for logs

4. **Versioning**:
   - Enable S3 bucket versioning for rollback capability

5. **Cost Optimization**:
   - Set a lifecycle rule in S3 to delete old versions after 30 days

## Final Notes

This setup gives you a production-ready frontend with automated deployments. The CI/CD pipeline ensures every push to main updates your site, and CloudFront's invalidation clears the cache for instant updates. Test each step carefully, especially DNS and SSL, as propagation can take time (up to 48 hours).
