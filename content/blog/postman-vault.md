+++
title = "Secure API Gateway with Amazon Cognito using SAM - PKCE Edition"
date = 2024-05-15T00:00:00-00:00
draft = false
description = "Last week I released a post on how to secure APIs using machine-to-machine authentication. Exactly one day after that AWS Cognito changed their prices where my proposed setup would actually incurr cost. In this post I will go through a different using the PKCE flow. This will still allow us to authenticate using Postman and will keep us in the free tier."
tags = ["AWS", "Security", "Serverless", "SAM"]
[[images]]
  src = "img/api-cognito-pkce/title.png"
  alt = ""
  stretch = "stretchH"
+++

1. Setup Vault
1. Created and saved vault key in 1password
1. can't save to native password manager with web version (try it out with desktop version next)
1. revoked vault key
1. created new one from desktop it was the same experience.
1. add 1password integration
1. create 1password service account
1. 