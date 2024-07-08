+++
title = "Creating a user in Amazon Cognito programmatically"
date = 2024-06-30T00:00:00-00:00
draft = false
description = "Short post to walk through the steps to create a user in Amazon Cognito to be used for automation"
tags = ["AWS", "Security", "Cognito"]
[[images]]
  src = "img/create-cognito-user-programmatically/title-es.png"
  alt = ""
  stretch = "stretchH"
+++

When you have CI/CD pipelines that run automated tests against your APIs you might need to dynamically create users in Amazon Cognito to run them. If that is the case you are in the right place. In this post we'll be going over what you need to do to create a valid user in Cognito to be used by your automation.

## Create user in Cognito
The first thing we need to do is to create a user in our Cognito User Pool. Since this is for automation purposes I am creating users using the *andmore.dev* domain that I own. I want these users scoped to the specific repository that is running the CI/CD pipeline, so I'm generating a username with a UUID.

```bash
username=$(uuidgen)@andmore.dev
aws cognito-idp admin-create-user --user-pool-id [YOUR-USERPOOL-ID] --username $username --message-action SUPPRESS
```
In the commands above, I am generating the username with new uuid, since I have email forwarding enabled for my domain, each new user will send me an email. To avoid this I have added the `--message-actions` property as `SUPPRESS`, this will disable any messages from being sent.

This is all we need right? I thought this too, but there is more. The user is created with a temporary password and is not usable yet. Since we are doing this for an automated process, we can't really go through the flow of opening the email, signing in and changing the password. So let's see what else we need to do.

## Verifying user
Cognito offers a command that allows admins to set a users password, bypassing the manual steps mentioned above.

```bash
password=$(uuidgen)G1%
aws cognito-idp admin-set-user-password --user-pool-id [YOUR-USERPOOL-ID] --username $username  --password $password --permanent
```
We first need to generate a password, to keep it simple I'm doing it by creating a new uuid and appending a capital letter, a number and a special character since my user pool requires this as part of the password policy that we set. Now we can provide this to the `admin-set-user-password` command and make sure we pass in the `--permanent` flag, since by default it will set the password as temporary therefore not allowing us to use it.

Now the user is all setup to be able to authenticate using any of the flows you have setup for your Cognito User Pool Client.

## Delete the user
You probably don't want to keep all of these users once your pipeline is done. To delete the user you will need to call the `admin-delete-user` command with the generated username. 
```bash
aws cognito-idp admin-delete-user --user-pool-id [YOUR-USERPOOL-ID] --username $username
```

## IAM Permissions
To be able to run these commands you'll need permissions to run these for your Cognito user pool. Below is an example IAM policy that will give you the necessary permissions to run all three commands. Make sure to restrict the access to this policy since with this level of access people could create users and potentially do harm in your product.
```json
{
  "Effect": "Allow",
  "Action": [
      "cognito-idp:AdminCreateUser",
      "cognito-idp:AdminSetUserPassword",
      "cognito-idp:AdminDeleteUser",
  ],
  "Resource": [
      "YOUR-USERPOOL-ARN",
  ]
}
```

## Wrap up
We were able to easily create and verify a user to be used by our automation, as well as clean up after we are done so we are not left with orphaned users all over the place.
I hope this is helpful to you. Cognito documentation is really not that straightforward and can be hard to find what you are looking for, this is why I'm trying to provide tutorials around Cognito, so stay tuned since I have more content lined up around this topic.