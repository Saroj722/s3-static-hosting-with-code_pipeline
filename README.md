# AWS Static Website Hosting with S3, CloudFront, CodeBuild & CodePipeline

This project demonstrates how to host a static website on **Amazon S3**, distribute it globally using **Amazon CloudFront**, and automate the build and deployment process using **AWS CodeBuild** and **AWS CodePipeline**.

The project also uses **Amazon Route 53** for DNS routing and **AWS Certificate Manager (ACM)** for HTTPS/SSL.

## Architecture

```text
                 ┌─────────────────┐
                 │     GitHub      │
                 │   Source Code   │
                 └────────┬────────┘
                          │
                          │ Webhook
                          ▼
                 ┌─────────────────┐
                 │  AWS CodeBuild  │
                 │     Build       │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │  AWS CodePipeline│
                 │   Deployment    │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │   Amazon S3     │
                 │ Static Website  │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │ Amazon CloudFront│
                 │  CDN + HTTPS    │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │   Route 53 DNS  │
                 │  Custom Domain  │
                 └─────────────────┘
```

## AWS Services Used

* **Amazon S3** – Static website hosting and deployment destination
* **Amazon CloudFront** – Content delivery, caching, and HTTPS
* **AWS CodeBuild** – Builds the application and prepares the deployment
* **AWS CodePipeline** – Automates the CI/CD workflow
* **Amazon Route 53** – DNS management
* **AWS Certificate Manager (ACM)** – SSL/TLS certificate
* **AWS WAF** – Web application security protections
* **GitHub** – Source code repository

## Prerequisites

Before starting, make sure you have:

* An AWS account
* A GitHub account
* A GitHub repository containing your static website
* A registered domain name
* Access to Amazon Route 53 if you want to use a custom domain
* An ACM certificate for your domain

---

# 1. Create an S3 Bucket

Create an S3 bucket to host the static website.

For example:

```text
github-project-s3-static-with-codebuild
```

Enable **Static Website Hosting** for the bucket.

Set the index document to:

```text
index.html
```

If your website has an error page, you can also configure:

```text
error.html
```

## S3 Bucket Policy

Update the bucket policy to allow CloudFront/public access to objects as required by your architecture.

Example:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::github-project-s3-static-with-codebuild/*"
        }
    ]
}
```

> **Important:** Replace the bucket ARN with your own S3 bucket ARN.

For production environments, consider using a **CloudFront Origin Access Control (OAC)** and keeping the S3 bucket private rather than allowing public access.

---

# 2. Create a CloudFront Distribution

Go to:

**AWS Console → CloudFront → Create Distribution**

Configure the distribution as follows.

### Distribution Name

For example:

```text
s3-static-hosting
```

### Origin

Select your S3 bucket as the origin.

For example:

```text
github-project-s3-static-with-codebuild
```

Set the default root object to:

```text
index.html
```

### WAF

Enable the available security protections if required.

### TLS / HTTPS

If you are using a custom domain such as:

```text
hi.example.com
```

create or select an appropriate SSL/TLS certificate from **AWS Certificate Manager (ACM)**.

> **Important:** CloudFront certificates must be created in the **US East (N. Virginia) / `us-east-1`** AWS Region.

Attach the certificate to the CloudFront distribution and add your custom domain under the distribution's alternate domain names.

Finally, create the distribution.

After creation, CloudFront will provide a **Distribution ID**, for example:

```text
E5U8I6USMQ22T
```

You will need this ID in the deployment configuration.

---

# 3. Configure AWS CodeBuild

Go to:

**AWS Console → CodeBuild → Build projects → Create build project**

Give the project a name, for example:

```text
s3-static-website-build
```

## Source

Select:

```text
GitHub
```

Connect your GitHub account if it has not already been connected.

Select the GitHub repository containing your website.

## Webhook

Enable webhook events so that changes pushed to GitHub can automatically trigger the build.

## Service Role

Select an existing CodeBuild service role or allow AWS to create a new role.

The CodeBuild service role needs the required permissions to perform the deployment.

For this project, make sure the role has the necessary permissions for:

* S3
* CloudFront
* CodeBuild

Avoid granting overly broad permissions in production. Use the minimum permissions required by your build and deployment process.

## Buildspec

Specify the location of your `buildspec.yml` file.

For example:

```text
buildspec.yml
```

The `buildspec.yml` file defines the commands CodeBuild executes during the build process.

---

# 4. Configure CodePipeline

Go to:

**AWS Console → CodePipeline → Create pipeline**

Give your pipeline a name, for example:

```text
s3-static-website-pipeline
```

## Pipeline Settings

Create a new service role if you don't already have one.

Click **Next**.

## Source Stage

Select:

```text
GitHub
```

Choose your GitHub connection and select the repository containing your project.

Enable the webhook/trigger so that changes pushed to GitHub automatically start the pipeline.

Click **Next**.

## Build Stage

Select:

```text
AWS CodeBuild
```

Choose the CodeBuild project created in the previous step.

For example:

```text
s3-static-website-build
```

Click **Next**.

## Deploy Stage

Select:

```text
Amazon S3
```

Choose the S3 bucket created earlier.

Enable:

```text
Extract file before deploy
```

This ensures the files generated by the pipeline are extracted into the S3 bucket rather than uploaded as a single archive.

Click **Next** and create the pipeline.

---

# 5. Automated CI/CD Workflow

After the pipeline has been created, the deployment process becomes automated.

The workflow is:

```text
Developer
    │
    │ git push
    ▼
GitHub
    │
    │ Webhook
    ▼
CodePipeline
    │
    ▼
CodeBuild
    │
    │ Build
    ▼
S3
    │
    │ Website files
    ▼
CloudFront
    │
    ▼
Custom Domain
```

Whenever you push changes to GitHub:

1. GitHub triggers the CodePipeline webhook.
2. CodePipeline retrieves the latest source code.
3. CodeBuild runs the `buildspec.yml`.
4. The application is built.
5. The generated files are deployed to the S3 bucket.
6. CloudFront serves the updated website to users.

---

# 6. Configure Route 53

To use your custom domain, go to:

**AWS Console → Route 53 → Hosted zones**

Select your domain.

Create an **A record** for the desired subdomain.

For example:

```text
hi.example.com
```

For the record type, select:

```text
A
```

Enable:

```text
Alias
```

Set the Alias target to your **CloudFront distribution**.

For example:

```text
E5U8I6USMQ22T
```

Save the record.

Your domain should now route traffic through CloudFront to your S3-hosted website.

---

# 7. Buildspec Configuration

The project uses a `buildspec.yml` file to define the CodeBuild process.

A basic example is:

```yaml
version: 0.2

phases:
  build:
    commands:
      - echo "Build started"
      - echo "Building website..."

  post_build:
    commands:
      - echo "Build completed"
      - aws cloudfront create-invalidation --distribution-id E5U8I6USMQ22T --paths "/*"

artifacts:
  files:
    - '**/*'
```

Replace:

```text
E5U8I6USMQ22T
```

with your own CloudFront Distribution ID.

> Modify the build commands according to your application. If your project requires `npm install`, `npm run build`, or another build process, add those commands to the appropriate CodeBuild phase.

---

# 8. Project Structure

A typical repository structure can look like this:

```text
.
├── index.html
├── css/
│   └── style.css
├── js/
│   └── script.js
├── images/
│   └── ...
├── buildspec.yml
└── README.md
```

If you are using a framework such as React, Angular, or Vue, your structure may be different and the `buildspec.yml` should be adjusted accordingly.

---

# 9. Deployment Flow

The complete deployment architecture is:

```text
GitHub
   │
   │ Push
   ▼
AWS CodePipeline
   │
   │ Source
   ▼
AWS CodeBuild
   │
   │ Build
   ▼
Amazon S3
   │
   │ Origin
   ▼
Amazon CloudFront
   │
   │ HTTPS
   ▼
Route 53
   │
   ▼
Custom Domain
```

This provides a simple **CI/CD pipeline for static website hosting on AWS**.

---

# 10. Key Features

* Static website hosting using Amazon S3
* Global content delivery using CloudFront
* HTTPS using AWS Certificate Manager
* DNS management using Route 53
* Automated builds using CodeBuild
* Automated deployment using CodePipeline
* GitHub integration
* Webhook-based deployments
* CloudFront cache invalidation after deployment
* Optional AWS WAF security protection

---

# 11. How to Deploy Changes

After the initial setup, deployment is simple.

Make changes to the project:

```bash
git add .
git commit -m "Update website"
git push origin main
```

The GitHub webhook will trigger the AWS CodePipeline automatically.

The pipeline will then:

```text
GitHub
   ↓
CodePipeline
   ↓
CodeBuild
   ↓
S3
   ↓
CloudFront
```

After the pipeline completes successfully, the updated website will be available through your configured domain.

---

# 12. Troubleshooting

### Website is not loading

Check:

* S3 bucket configuration
* S3 bucket policy
* CloudFront distribution status
* CloudFront origin configuration
* Route 53 DNS record
* ACM certificate status

### Changes are not visible

CloudFront may still be serving cached content.

Create a CloudFront invalidation:

```bash
aws cloudfront create-invalidation \
  --distribution-id YOUR_DISTRIBUTION_ID \
  --paths "/*"
```

### CodePipeline is not triggered

Check:

* GitHub connection
* Webhook configuration
* Repository and branch settings
* CodePipeline source configuration

### CodeBuild fails

Check:

* `buildspec.yml`
* CodeBuild logs
* IAM permissions
* Build commands
* Runtime/environment configuration

---

# 13. Security Considerations

For learning purposes, the S3 bucket can be configured for public access as shown above.

For a production deployment, a more secure architecture is recommended:

```text
User
  │
  ▼
CloudFront
  │
  │ Origin Access Control
  ▼
Private S3 Bucket
```

With this architecture:

* S3 objects are not directly publicly accessible.
* Users access the website through CloudFront.
* CloudFront uses Origin Access Control to access S3.
* HTTPS is enforced at CloudFront.
* AWS WAF can provide additional protection.

---

# Conclusion

This project demonstrates how to build a complete AWS-based CI/CD pipeline for a static website.

The combination of **GitHub, CodePipeline, CodeBuild, S3, CloudFront, Route 53, ACM, and WAF** provides an automated workflow where code changes pushed to GitHub can be built and deployed automatically to a globally distributed HTTPS-enabled website.

## Technologies

```text
GitHub
AWS CodePipeline
AWS CodeBuild
Amazon S3
Amazon CloudFront
Amazon Route 53
AWS Certificate Manager
AWS WAF
```

## Author

Built as an AWS CI/CD and static website hosting project.
