---
layout: post
title: Creating and Adding AWS Accounts to Spinnaker (Deployment Target)
order: 32
# Change this to true when ready to publish
published: true
redirect_from:
  - /spinnaker_install_admin_guides/add-aws-account/
  - /spinnaker_install_admin_guides/add_aws_account/
  - /spinnaker-install-admin-guides/add_aws_account/
---

Once you have (OSS or Armory) Spinnaker up and running in Kubernetes, you'll want to start adding deployment targets.  *(This document assumes Spinnaker was installed with halyard, that you have access to your current halconfig (and a way to operate `hal` against it), and that you have a kubeconfig and `kubectl` with permissions to create the relevant Kubernetes entities (`service account`, `role`, and `rolebinding`)*

* This is a placeholder for an unordered list that will be replaced with ToC. To exclude a header, add {:.no_toc} after it.
{:toc}

## Overview

This document will guide you through one of the following:

* Configuring Spinnaker to use AWS IAM Instance Roles (if Spinnaker is running on AWS, either via AWS EKS or installed directly on EC2 instances)
  * Creating a Managed Account IAM Role in each your target AWS Accounts
  * Creating a Managing Account IAM Policy in your primary AWS Account
  * Adding the Managing Account IAM Policy to the existing IAM Instance Role on the AWS nodes
  * Configuring the Managed Accounts IAM Roles to trust the IAM Instance Role from the AWS nodes
  * Adding the Managed Accounts to Spinnaker via Halyard
  * Adding/Enabling the AWS Cloudprovider to Spinnaker
* Configuring Spinnaker to access AWS using an IAM User (with an Acess Key and Secret Access Key)
  * Creating a Managed Account IAM Role in each your target AWS Accounts
  * Creating a Managing Account IAM Policy in your primary AWS Account
  * Creating a Managing Account IAM User with access to the Managing Account Policy
  * Configuring the Managed Accounts to trust the Managing Account IAM User
  * Adding the Managing Account User and Managed Accounts to Spinnaker via Halyard
  * Adding/Enabling the AWS Cloudprovider to Spinnaker

_This document is intentionally redundant in order to support end-to-end workflows, rather than mixing and matching steps._

## Background

Even though Spinnaker is installed in Kubernetes, it can be used to deploy to other cloud environments, such as AWS.  Rather than granting Spinnaker direct access to each of the target AWS accounts, Spinnaker will assume a role in each of the target accounts.  This behaves roughly as follows:

* Spinnaker's Clouddriver Pod should be able to assume a **Managed Account Role** in each deployment target AWS account, and use that role to perform any AWS actions.  This may include one or more of the following:
  * Create AWS Launch Configurations and Auto Scaling Groups to deploy AWS EC2 instances
  * Run ECS Containers
  * Run AWS Lambda Actions (alpha/beta as of the time of this document)
  * Create AWS CloudFormation Stacks (alpha/beta as of the time of this document)
* Clouddriver is configured with direct access to a **"Managing Account"** Policy (_it may be helpful to think of this as the **Master** or **Source** Policy_), which is accomplished on one of two ways:
  * If Spinnaker is running in AWS (either in AWS EKS, or with Kubernetes nodes running in AWS EC2), the Managing Account Policy can be made available to Spinnaker by adding it to the AWS nodes (EC2 instances) where the Spinnaker Clouddriver pod(s) are running.
    * _(You can also use Kube2IAM or similar capabilities, but this is not covered in this document)_
  * An IAM User with access to the Managing Account Policy can be passed directly to Spinnaker via an Access Key and Secret Access Key, configured via Halyard
* For each AWS account that you want Spinnaker to be able to deploy to, Spinnaker needs a **"Managed Account"** Role in that AWS account, with permissions to do the things you want Spinnaker to be able to do (_it may be helpful to think of this as a **Target Role**_)
* The Managing Account Role (Source/Master Role) should be able to assume each of the Managed Account Roles (Target Roles).  This requires two things:
  * The Managing Account Role needs a permission string for each Managed Account it needs to be able to assume.  _It may be helpful to think of this as an outbound permission._
  * Each Managed Account needs to have a trust relationship with the Managing Account User or Role to allow the Managing Account User or Role to assume it.  _It may be helpful to think of this as an inbound permission._

## Prerequisites

This document assumes the following:

* Your Spinnaker is up and running
* Your Spinnaker was installed and configured via Halyard
* You have access to Halyard
* You have permissions to create IAM roles using IAM policies and permissions, in all relevant AWS accounts
  * You should also be able to set up cross-account trust relationships between IAM roles.
* If you want to add the IAM Role to Spinnaker via an Access Key/Secret Access Key, you have permissions to create an IAM User
* If you want to add the IAM Role to Spinnaker via IAM instance profiles/policies, you have permissions to modify the IAM instance 

_All configuration with AWS in this document will be handled via the browser-based AWS Console.  All configurations could **alternately** be configured via the `aws` CLI, but this is not currently covered in this document._

Also - we will be granting AWS Power User Access to each of the Managed Account Roles.  You could optionally grant fewer permisisons, but those more limited permissions are not covered in this document.

## Configuring Spinnaker to use AWS IAM Instance Roles

If you are running Spinnaker on AWS (either via AWS EKS or installed directly on EC2 instances), you can use AWS IAM roles to allow Clouddriver to interact with the various AWS APIs across multiple AWS Accounts.

### Instance Role Part 1: Creating a Managed Account IAM Role in each your target AWS Accounts

In each account that you want Spinnaker to deploy to, you should create an IAM role for Spinnaker to assume.

For each account you want to deploy to, perform the following:
1. Log into the browser-based AWS Console
1. Navigate to the IAM page (click on "Services" at the top, then on "IAM" under "Security, Identity, & Compliance")
1. Click on "Roles" on the left hand side
1. Click on "Create role"
1. For now, for the "Choose the service that will use this role", select "EC2".  We will change this later, because we want to specify an explicit consumer of this role later on.
1. Click on "Next: Permissions"
1. Search for "PowerUserAccess" in the search filter, and select the Policy called "PowerUserAcces"
1. Click "Next: Tags"
1. Optionally, add tags that will identify this role.
1. Click "Next: Review"
1. Enter a Role Name.  For example, "DevSpinnakerManagedRole".  Optionally, add a description, such as "Allows Spinnaker Dev Cluster to perform actions in this account."
1. Click "Create Role"
1. In the list of Roles, click on your new Role (you may have to scroll down or filter for it).
1. Copy the Role ARN and save it.  It should look something like this: `arn:aws:iam::123456789012:role/DevSpinnakerManagedRole`.  **This will be used in the next section, "Instance Role Part 2", and in the Halyard section, "Instance Role Part 5"***

You will end up with a Role ARN for each Managed / Target account.  The Role names do not have to be the same (although it is a bit cleaner if they are).  For example, you may end up with roles that look like this:
* `arn:aws:iam::123456789012:role/DevSpinnakerManagedRole`
* `arn:aws:iam::123456789013:role/DevSpinnakerManagedRole`
* `arn:aws:iam::123456789014:role/DevSpinnakerManaged`

### Instance Role Part 2: Creating a Managing Account IAM Policy in your primary AWS Account

In the account that Spinnaker lives in (i.e., the AWS account that owns the EKS cluster where Spinnaker is installed), create an IAM Policy with permissions to assume all of your Managed Roles.

1. Log into the AWS account where Spinnaker lives, into the browser-based AWS Console
1. Navigate to the IAM page (click on "Services" at the top, then on "IAM" under "Security, Identity, & Compliance")
1. Click on "Policies" on the left hand side
1. Click on "Create Policy"
1. Click on the "JSON" tab, and paste in this:
  ```json
  {
      "Version": "2012-10-17",
      "Statement": [
          {
              "Effect": "Allow",
              "Action": [
                  "ec2:DescribeAvailabilityZones",
                  "ec2:DescribeRegions"
              ],
              "Resource": [
                  "*"
              ]
          },
          {
              "Action": "sts:AssumeRole",
              "Resource": [
                  "arn:aws:iam::123456789012:role/DevSpinnakerManagedRole",
                  "arn:aws:iam::123456789013:role/spinnakerManaged",
                  "arn:aws:iam::123456789014:role/DevSpinnakerManaged"
              ],
              "Effect": "Allow"
          }
      ]
  }
  ```
1. Update the `sts:AssumeRole` block with the list of Managed Roles you created in **Instance Role Part 1**.
1. Click on "Review Policy"
1. Create a name for your policy, such as "DevSpinnakerManagingPolicy".  Optionally, add a descriptive description.  Copy the name of the policy.  **This will be used in the next section, "Instance Role Part 3"***
1. On the list policies, click your newly-created Policy.

_(This policy could also be attached inline directly to the IAM Instance Role, rather than creating a standalone policy)_

### Instance Role Part 3: Adding the Managing Account IAM Policy to the existing IAM Instance Role on the AWS nodes

1. Log into the AWS account where Spinnaker lives, into the browser-based AWS Console
1. Navigate to the EC2 page (click on "Services" at the top, then on "EC2" under "Compute")
1. Click on "Running Instances"
1. Find one of the nodes which is part of your EKS or other Kubernetes cluster, and select it.
1. In the Instance details section of the screen (in the lower half), find the "IAM Role" and click on it to go to the Role page.
1. Click on "Attach Policies"
1. Search for the Policy that you created, and select it.
1. Click "Attach Policy"
1. Back on the screen for the Role, copy the node role ARN.  It should look something like this: `arn:aws:iam::123456789010:role/node-role`.  **This will be used in the next section, "Instance Role Part 4"**

### Instance Role Part 4: Configuring the Managed Accounts IAM Roles to trust the IAM Instance Role from the AWS nodes

Now that we know what role will be assuming each of the Managed Roles, we must configure the Managed Roles (Target Roles) to trust and allow the Managing (Assuming) Role to assume them.  This is called a "Trust Relationship" and is configured each of the Managed Roles (Target Roles).

For each account you want to deploy to, perform the following:
1. Log into the browser-based AWS Console
1. Navigate to the IAM page (click on "Services" at the top, then on "IAM" under "Security, Identity, & Compliance")
1. Click on "Roles" on the left hand side
1. Find the Managed Role that you created earlier in this account, and click on the Role Name to edit the role.
1. Click on the "Trust relationships" tab.
1. Click on "Edit trust relationship"
1. Replace the Policy Document with this (Update the ARN with the node role ARN from "Instance Role Part 3")

    ```json
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Principal": {
            "AWS": [
              "arn:aws:iam::123456789010:role/node-role"
            ]
          },
          "Action": "sts:AssumeRole"
        }
      ]
    }
    ```

8. Click "Update Trust Policy", in the bottom right.

### Instance Role Part 5: Adding the Managed Accounts to Spinnaker via Halyard

The Clouddriver pod(s) should be now able to assume each of the Managed Roles (Target Roles) in each of your Deployment Target accounts.  We need to configure Spinnaker to be aware of the accounts and roles its allowed to consume.  This is done via Halyard.

For each of the Managed (Target) accounts you want to deploy to, perform the following from your Halyard instance:

1. Run this command, **updating fields as follows**:
    * `AWS_ACCOUNT_NAME` should be a unique name which is used in the Spinnaker UI and API to identify the deployment target.  For example, `aws-dev-1` or `aws-dev-2`
    * `ACCOUNT_ID` should be the account ID for the Managed Role (Target Role) you are assuming.  For example, if the role ARN is `arn:aws:iam::123456789012:role/DevSpinnakerManagedRole`, then ACCOUNT_ID would be `123456789012`
    * `ROLE_NAME` should be the full role name within the account, including the type of object (`role`).  For example, if the role ARN is `arn:aws:iam::123456789012:role/DevSpinnakerManagedRole`, then ROLE_NAME would be `role/DevSpinnakerManagedRole`
  
    ```bash
    # Enter the account name you want Spinnaker to use to identify the deployment target, the account ID, and the role name.
    export AWS_ACCOUNT_NAME=aws-dev-1
    export ACCOUNT_ID=123456789012
    export ROLE_NAME=role/DevSpinnakerManagedRole

    hal config provider aws account add ${AWS_ACCOUNT_NAME} \
        --account-id ${ACCOUNT_ID} \
        --assume-role ${ROLE_NAME}
    ```

1. Optionally, edit the account with additional options such as those indicated in the [halyard documentation](https://www.spinnaker.io/reference/halyard/commands/#hal-config-provider-aws-account-edit).  For example, to set the regions that you can deploy to:

    ```bash
    export AWS_ACCOUNT_NAME=aws-dev-1
    hal config provider aws account edit ${AWS_ACCOUNT_NAME} \
        --regions us-east-1,us-west-2
    ```

### Instance Role Part 6: Adding/Enabling the AWS Cloudprovider to Spinnaker

Once you've added all of the Managed (Target) accounts, run these commands to set up and enable the AWS cloudprovider setting as whole (this can be run multiple times with no ill effects):

1. Enable the AWS Provider
  ```bash
  hal config provider aws enable
  ```
1. Apply all Spinnaker changes:
  ```bash
  # Apply changes
  hal deploy apply
  ```

## Configuring Spinnaker to access AWS using an IAM User (with an Acess Key and Secret Access Key)

If you are not running Spinnaker on AWS, or if you don't want to use AWS IAM roles (or don't have the ability to modify the roles attached to your Kubernetes instances), you can create an AWS IAM user and provide its credentials to Clouddriver to allow Clouddriver to interact with the various AWS APIs across multiple AWS Accounts.

### IAM User Part 1: Creating a Managed Account IAM Role in each your target AWS Accounts

In each account that you want Spinnaker to deploy to, you should create an IAM role for Spinnaker to assume.

For each account you want to deploy to, perform the following:

1. Log into the browser-based AWS Console
1. Navigate to the IAM page (click on "Services" at the top, then on "IAM" under "Security, Identity, & Compliance")
1. Click on "Roles" on the left hand side
1. Click on "Create role"
1. For now, for the "Choose the service that will use this role", select "EC2".  We will change this later, because we want to specify an explicit consumer of this role later on.
1. Click on "Next: Permissions"
1. Search for "PowerUserAccess" in the search filter, and select the Policy called "PowerUserAcces"
1. Click "Next: Tags"
1. Optionally, add tags that will identify this role.
1. Click "Next: Review"
1. Enter a Role Name.  For example, "DevSpinnakerManagedRole".  Optionally, add a description, such as "Allows Spinnaker Dev Cluster to perform actions in this account."
1. Click "Create Role"
1. In the list of Roles, click on your new Role (you may have to scroll down or filter for it).
1. Copy the Role ARN and save it.  It should look something like this: `arn:aws:iam::123456789012:role/DevSpinnakerManagedRole`.  **This will be used in the next section, "IAM User Part 2", and in the Halyard section, "IAM User Part 5"***

You will end up with a Role ARN for each Managed / Target account.  The Role names do not have to be the same (although it is a bit cleaner if they are).  For example, you may end up with roles that look like this:

* `arn:aws:iam::123456789012:role/DevSpinnakerManagedRole`
* `arn:aws:iam::123456789013:role/DevSpinnakerManagedRole`
* `arn:aws:iam::123456789014:role/DevSpinnakerManaged`

### IAM User Part 2: Creating a Managing Account IAM Policy in your primary AWS Account

In the account that Spinnaker lives in (i.e., the AWS account that owns the EKS cluster where Spinnaker is installed), create an IAM Policy with permissions to assume all of your Managed Roles.

1. Log into the AWS account where Spinnaker lives, into the browser-based AWS Console
1. Navigate to the IAM page (click on "Services" at the top, then on "IAM" under "Security, Identity, & Compliance")
1. Click on "Policies" on the left hand side
1. Click on "Create Policy"
1. Click on the "JSON" tab, and paste in this:
  ```json
  {
      "Version": "2012-10-17",
      "Statement": [
          {
              "Effect": "Allow",
              "Action": [
                  "ec2:DescribeAvailabilityZones",
                  "ec2:DescribeRegions"
              ],
              "Resource": [
                  "*"
              ]
          },
          {
              "Action": "sts:AssumeRole",
              "Resource": [
                  "arn:aws:iam::123456789012:role/DevSpinnakerManagedRole",
                  "arn:aws:iam::123456789013:role/spinnakerManaged",
                  "arn:aws:iam::123456789014:role/DevSpinnakerManaged"
              ],
              "Effect": "Allow"
          }
      ]
  }
  ```
1. Update the `sts:AssumeRole` block with the list of Managed Roles you created in **IAM User Part 1**.
1. Click on "Review Policy"
1. Create a name for your policy, such as "DevSpinnakerManagingPolicy".  Optionally, add a descriptive description.  Copy the name of the policy.  **This will be used in the next section, "IAM User Part 3"**
1. On the list policies, click your newly-created Policy.

_(This policy could also be attached inline directly to the IAM User, rather than creating a standalone policy)_

### IAM User Part 3: Creating a Managing Account IAM User with access to the Managing Account Policy

The IAM user we're creating can be in any AWS account, although it may make sense to place it in the same account where Spinnaker lives if Spinnaker is installed in AWS.

1. Log into the AWS account where Spinnaker lives, into the browser-based AWS Console
1. Navigate to the IAM page (click on "Services" at the top, then on "IAM" under "Security, Identity, & Compliance")
1. Click on "Users" on the left side
1. Click on "Add user"
1. Specify a logical user name, such as "DevSpinnakerManagingAccount"
1. Check the "Programmatic access" checkbox
1. Select "Attach existing policies directly"
1. Find the policy you created in "IAM User Part 2". and select it with the checkbox.
1. Click "Next: Tags"
1. Optionally, add tags that will identify this user
1. Click "Next: Review"
1. Click "Create user"
1. Copy the "Access key ID" and "Secret access key" (you'll have to click "Show").  **This will be used later, in "IAM User Part 6"**
1. Click "Close"
1. Click on the "User name" for the user that you just created.
1. Copy the "User ARN".  This will look something like this: `arn:aws:iam::123456789010:user/DevSpinnakerManagingAccount`.  **This will be used in the next section, "IAM User Part 4"**

### IAM User Part 4: Configuring the Managed Accounts to trust the Managing Account IAM User

Now that we know what user will be assuming each of the Managed Roles, we must configure the Managed Roles (Target Roles) to trust and allow the Managing (Assuming) User to assume them.  This is called a "Trust Relationship" and is configured each of the Managed Roles (Target Roles).

For each account you want to deploy to, perform the following:
1. Log into the browser-based AWS Console
1. Navigate to the IAM page (click on "Services" at the top, then on "IAM" under "Security, Identity, & Compliance")
1. Click on "Roles" on the left hand side
1. Find the Managed Role that you created earlier in this account, and click on the Role Name to edit the role.
1. Click on the "Trust relationships" tab.
1. Click on "Edit trust relationship"
1. Replace the Policy Document with this (Update the ARN with the node role ARN from "Instance Role Part 3")

    ```json
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Principal": {
            "AWS": [
              "arn:aws:iam::123456789010:user/DevSpinnakerManagingAccount",
            ]
          },
          "Action": "sts:AssumeRole"
        }
      ]
    }
    ```

1. Click "Update Trust Policy", in the bottom right.

### IAM User Part 5: Adding the Managing Account User and Managed Accounts to Spinnaker via Halyard

The Clouddriver pod(s) should be now able to assume each of the Managed Roles (Target Roles) in each of your Deployment Target accounts.  We need to configure Spinnaker to be aware of the accounts and roles its allowed to consume.  This is done via Halyard.

For each of the Managed (Target) accounts you want to deploy to, perform the following from your Halyard instance:
1. Run this command, **updating fields as follows**:

    * `AWS_ACCOUNT_NAME` should be a unique name which is used in the Spinnaker UI and API to identify the deployment target.  For example, `aws-dev-1` or `aws-dev-2`
    * `ACCOUNT_ID` should be the account ID for the Managed Role (Target Role) you are assuming.  For example, if the role ARN is `arn:aws:iam::123456789012:role/DevSpinnakerManagedRole`, then ACCOUNT_ID would be `123456789012`
    * `ROLE_NAME` should be the full role name within the account, including the type of object (`role`).  For example, if the role ARN is `arn:aws:iam::123456789012:role/DevSpinnakerManagedRole`, then ROLE_NAME would be `role/DevSpinnakerManagedRole`

    ```bash
    # Enter the account name you want Spinnaker to use to identify the deployment target, the account ID, and the role name.
    export AWS_ACCOUNT_NAME=aws-dev-1
    export ACCOUNT_ID=123456789012
    export ROLE_NAME=role/DevSpinnakerManagedRole

    hal config provider aws account add ${AWS_ACCOUNT_NAME} \
        --account-id ${ACCOUNT_ID} \
        --assume-role ${ROLE_NAME}
    ```

1. Optionally, edit the account with additional options such as those indicated in the [halyard documentation](https://www.spinnaker.io/reference/halyard/commands/#hal-config-provider-aws-account-edit).  For example, to set the regions that you can deploy to:

    ```bash
    export AWS_ACCOUNT_NAME=aws-dev-1
    hal config provider aws account edit ${AWS_ACCOUNT_NAME} \
        --regions us-east-1,us-west-2
    ```

### IAM User Part 6: Adding/Enabling the AWS Cloudprovider to Spinnaker

Once you've added all of the Managed (Target) accounts, run these commands to set up and enable the AWS cloudprovider setting as whole (this can be run multiple times with no ill effects):

1. Add the AWS access key and secret access key from "IAM User Part 3" using Halyard (don't forget to provide the correct access key).
  
    ```bash
    export ACCESS_KEY_ID=AKIA1234567890ABCDEF
    hal config provider aws edit --access-key-id ${ACCESS_KEY_ID} \
      --secret-access-key # do not supply the key here, you will be prompted
    ```

1. Enable AWS Provider

    ```bash
    hal config provider aws enable
    ```

1. Apply all Spinnaker changes:

    ```bash
    # Apply changes
    hal deploy apply
    ```