---
title: Hosting a Static Website on an S3 Bucket
date: 2024-03-07 10:49 -0500
category: cloud
tags: [aws, s3, iam, ci/cd]
---
In this post I'd like to talk a bit about my experience trying to host a static site from an S3 bucket in AWS. For the most part, this project wasn't much different from setting up a static website on a VPS---the major difference being the deploy action of the CI/CD pipeline. This post will not be a tutorial, just a discussion about challenges I encountered and basic steps to accomplish the task at hand.

## Creating the Site

You can read more about the goals of the website project [here](https://github.com/chase-slept/study-log), but the gist is that I wanted to set up a site to practice some AWS basics: in this case, hosting a website via S3 and setting up IAM credentials. The site design/build itself didn't really matter towards this goal, so I used a SSG (static site generator) to build it and add new posts. I picked out a simple theme, rather than spend time building a layout from scratch. To create the site and its files, I followed the instructions to fork the theme's project files and then set up a local development environment using that repository.

Once the site generator was set up locally, the first problem to solve was how to deploy the site right onto an S3 bucket. The theme I selected already had a CI/CD workflow included but I'd need to modify it to work with S3. Before I could modify the build and deploy actions, I would need to set up the AWS services and create credentials for the Github Actions runner.

## Creating and Configuring an S3 Bucket

In the AWS Console, I created an S3 bucket and enabled 'Static website hosting' in the bucket properties. Enabling this generated a bucket endpoint URL which I saved in a notepad for later. By default, S3 buckets are not publicly accessible---I needed to uncheck the "Block all public access" checkbox in the bucket properties and explicitly enable public access in the bucket permissions via a bucket policy like this:

{% highlight json %}
{
    "Version": "2024-02-28",
    "Id": "PolicyForPublicWebsiteContent",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": {
                "AWS": "*"
            },
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::<bucket_name>/*"
        }
    ]
}
{% endhighlight %}

## Creating an IAM User and Security Credentials

Next I needed to make a new user in IAM to allow the Github runner to access my AWS resources. I named the user appropriately, created access keys (and saved them for later!), and attached a permissions policy allowing limited access to S3 with the following parameters:

- Read:
  - GetBucketLocation
  - GetObject
- Write:
  - DeleteObject
  - PutBucketWebsite
  - PutObject
- List:
  - ListBucket
- Permission Management:
  - PutObjectAcl

*(I'll need to review the documentation for some of these options again later, but for now this got things working.)*

With the user created (login credentials saved for later) and the S3 bucket configured for public access, I could now set up the build/deploy action in Github.

## Configure the Github Action

I'll focus on just the AWS-relevant section of the Github Actions workflow, but here's a brief overview:

- code checkout
- install node and pnpm
- configure pnpm store path and cache
- install pnpm dependencies
- check astro for issues
- build/test site
- set AWS credentials  #the relevant bit
- deploy to S3  #also relevant

To configure the AWS credentials within the Github runner, I needed to make use of the [configure-aws-credentials](https://github.com/aws-actions/configure-aws-credentials/tree/v4/) Github Action. In the `with` section I reference the access keys via two secret variables (`AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`) and set the `aws-region` to match the S3 bucket.

{% highlight yaml %}
{% raw %}

- name: Set AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: us-east-2
{% endraw %}
{% endhighlight %}

Next, I configured the deploy action with the relevant AWS CLI command:

{% highlight yaml %}

- name: Deploy to S3
  run: aws s3 sync dist/ s3://<bucket_name> --delete
{% endhighlight %}

Lastly, I saved the access keys as secret variables in the Github repository's settings. With all of this configured, whenever new code is pushed to the repository (such as a new blog post) it will automatically be built, tested, and deployed to an S3 bucket. Pretty neat!
