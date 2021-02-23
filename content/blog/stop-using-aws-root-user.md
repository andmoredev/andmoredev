+++
title = "Stop using your AWS Account root user"
date = 2021-02-22T13:56:07-06:00
draft = true
description = "Really, stop it! You are putting your account at risk."
tags = ["AWS", "Security", "IAM"]
[[images]]
  src = "img/2021/02/pic03.jpg"
  alt = "Padlock"
  stretch = "stretchH"
+++

When you create a new AWS account it will create a *root user* with the email and password used to create it. The simplest thing to do is to use that user for everyday tasks. We will be looking at why you shouldn't do that and the configuration necessary to secure your account.

The *root user* has access to every AWS service and resource in an account. If the credentials for the root account are stolen they will be able to access or change anything in the account giving them the ability to misuse any data or resources in the account, they could even make you incurr unnecessary cost by creating resources in your account.

## Recommended settings for root user
Since it's critical to keep the root user safe and out of reach of malicious users there are a few things you can do to increase it's security.

* Do not create access keys for the root user. Create an IAM user for yourself with adminsitrative permissions
* Never share the root user credentials.
* Use a strong password. (Use a password manager if possible)
* Enable multi-factor authentication.

## When should you use the root user?
To handle day to day tasks and access to AWS resources you should use an [IAM user with the appropriate permissions](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html#lock-away-credentials) following the Principle Of Least Privilege (POLP).

There are a few actions that are only restricted to the root user, these are things such as:
* Updating the account name
* Changing credentials for the account
* Restore IAM user permissions (only when a single IAM administrator revokes their own permission)
* Closing your AWS account

[Here is a complete list](https://docs.aws.amazon.com/general/latest/gr/root-vs-iam.html#aws_tasks-that-require-root)

## Create an Administrator IAM user for yourself
It is very simple to get an IAM user setup for yourself in the AWS console to start working in your account.

Follow these steps and you will be good to go:

1. Log in to your AWS Account using your root user credentials (this should be the last time your using it unless you are doing one of the tasks restricted to the root user)
2. Go to IAM Service
3. In the users section press Add User
4. Enter `administrator` as the username
5. Select Programatic access and AWS Management Console access for Access Types.
6. Enter a strong password.
7. Remove the checkbox to require a password reset and click next.
![Add User Details](/img/2021/02/pic01.jpg)
8. Select the AdministratorAccess policy.
![Administrator User Policy](/img/2021/02/pic02.jpg)
9. Continue on the next steps to get the user created.

After the user is created you will be presented with the Access key ID and the Secret access key to be used for programmatic access.
Store these in a safe place to be used when using the AWS CLI

From now on this is the user you should use for day to day operations in your AWS account.

## Conclusion
Using your root user account for everyday tasks may be putting you at risk of an attack, by following a few simple steps you can reduce your attack exposure while still being able to do everything that you need in your account.
