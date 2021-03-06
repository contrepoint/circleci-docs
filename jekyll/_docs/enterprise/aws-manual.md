---
layout: enterprise
title: "Manual AWS Installation"
category: [resources]
order: 3
description: "How to install CircleCI Enterprise manually on Amazon Web Services (AWS)."
---

If you prefer granular control of all your own AWS resources, you can follow the following steps to manually launch your AWS resources:

## 1: Create an IAM user or role for CircleCI Enterprise machines
The instances running CircleCI Enterprise will require some access to S3 and EC2 API endpoints. CircleCI stores artifacts and caches in S3 and uses EC2 API for cluster purposes. (EC2 API access is optional, but improves builder machine status updates).  As a first approximation, we prefer the following IAM profile:

```
{
   "Version": "2012-10-17",
   "Statement" : [
      {
         "Action" : ["s3:*"],
         "Effect" : "Allow",
         "Resource" : [
            "arn:aws:s3:::circleci-*",
            "arn:aws:s3:::circleci-*/*"
         ]
      },
      {
          "Action": [
              "ec2:Describe*",
              "ec2:CreateTags",
              "cloudwatch:*",
              "iam:GetUser"
          ],
          "Resource": ["*"],
          "Effect": "Allow"
      }
   ]
}
```

To create a user with this policy, you can run the following commands using `awscli`:

```
$ aws iam create-user --user-name circleci-user
$ curl -sSL -o circleci-iam-policy.json https://enterprise-docs.circleci.com/assets/aws/circleci-iam-policy.json
$ cat circleci-iam-policy.json # please inspect the policy and validate the permissions it's using
$ aws iam create-policy --policy-name circleci-iam-policy \
    --policy-document file://circleci-iam-policy.json \
    --description "Policy to be used by CircleCI Enterprise machines"
[..You will get a large output with containing an Arn line... e.g..]
        "Arn": "arn:aws:iam::323423423452:policy/circleci-iam-policy",

$ aws iam attach-user-policy --user-name circleci-user --policy-arn <<policy-arn-from-last-output>>
$ aws iam create-access-key --user-name circleci-user
[.. Save the generated access key and secret for use in installation process..]
```

There will be an opportunity later on to provide the access key/secret associated with this user to the CircleCI application. Alternatively,
you can use an IAM role and instance profile with the same permissions.


## 2: Launch the CircleCI Enterprise services AMI

We recommend `c4.2xlarge` or `c4.4xlarge` for the services box, with an
EBS root volume of at least 30GB.

Ideally, the services box is placed in a security group with the following:

* whitelist ports 22, 80, 443, 8800 (used for system administration) to potential users
* whitelist all traffic for builder boxes

To install the server:

Start one of the following AMIs (for the appropriate AWS region), with an appropriately sized EBS root volume (~30 GB):

Region             | AMI
-----------------  |-------------
ap-northeast-1     | ami-c6293ca8
ap-northeast-2     | ami-38814956
ap-southeast-1     | ami-85ee3be6
ap-southeast-2     | ami-dcddfdbf
eu-central-1       | ami-175ebf78
eu-west-1          | ami-1955d16a
sa-east-1          | ami-a723accb
us-east-1          | ami-edc7cb87
us-west-1          | ami-ade597cd
us-west-2          | ami-934ca4f3
{: class="table table-striped"}

These AMIs are public and are marked as community AMIs.


## 3: Launch at least one CircleCI Enterprise builder AMI {#builder}

We recommend using beefy instances to allow for multiple concurrent builds. On the CircleCI hosted offering we use `r3.8xlarge` to run 10-15 builds concurrently.  Running with a smaller instances (e.g. `r3.4xlarge`) should fine as well.  The limiting factor is typically memory, as we default to allocating 4GB per build container.  You can start as many builder boxes as appropriate depending on how many concurrent builds or degree of parallelism you desire.

In our experience, spot instances make excellent builder boxes.

In terms of security groups:

* whitelist all traffic from/to builder boxes and service box
* whitelist ports 22, 80, 443, 64535-65535 for users.  The high ports are used for the SSH feature, so that users can ssh into the build containers.

To start a box:  Start a new instance with the appropriate region-specific AMI:

Region             | AMI
-----------------  |-------------
ap-northeast-1     | ami-c7c838a6
ap-northeast-2     | ami-b477bcda
ap-southeast-1     | ami-5775a734
ap-southeast-2     | ami-654e6506
eu-central-1       | ami-3401ea5b
eu-west-1          | ami-2237ad51
sa-east-1          | ami-8360f5ef
us-east-1          | ami-564df541
us-west-1          | ami-0b4a0d6b
us-west-2          | ami-b1a667d1
{: class="table table-striped"}

These AMIs are public and are marked as community AMIs.

SSH into the box as `ubuntu` user with the appropriate ssh key you set for the instance, and run

```
curl https://s3.amazonaws.com/circleci-enterprise/init-builder-0.2.sh | \
  SERVICES_PRIVATE_IP=<private ip address of services box> \
  CIRCLE_SECRET_PASSPHRASE=<passphrase entered on system console (services box port 8800) settings> \
  bash
```

The `sudo -H -u ubuntu` isn't strictly necessary when logged in as the "ubuntu"
user, but it allows this snippet to be used as-is in a user data script, which
is run as root.
It should take 5-15 minutes for the machine to be fully ready to take builds.
