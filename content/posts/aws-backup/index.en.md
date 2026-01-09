---
weight: 2
title: "AWS Backup -> Encryption Challenges"
date: 2025-12-07T00:00:00+00:00
lastmod: 2025-12-07T00:00:00+00:00
draft: false
author: "Jack"
authorLink: "https://jackroberts.uk"
description: "Understanding the different encryption challenges when using AWS Backup"
images: []

tags: ["aws", "backup", "ransomware", "data", "recovery", "cloud", "disaster recovery"]
categories: ["post"]

lightgallery: true

toc:
  auto: false
---

## Introduction

AWS Backup is a great solution for backing up supported AWS resources, allowing you to use AWS-native tools and resources to safeguard stateful, supported AWS resources. 

You'll notice above, I use the word "supported" a number of times. That is because AWS Backup does not support backup for _all_ stateful resources out-of-the-box. See below for AWS Backup Feature Availability: https://docs.aws.amazon.com/aws-backup/latest/devguide/backup-feature-availability.html 

The good news is that AWS Backup does support most of the "big ones", as you should hopefully see from table above. There is a catch, however. And it's this: amongst the resources that are supported, there is a difference in "feature availability" from one resource type to the next. For example, what is supported for a DynamoDB Table versus an RDS. And even within that - for example, Aurora has different "feature availability" to other RDS engines.

Now, what does this mean in practise? Ultimately you **will** be able to backup your resources, but the how/what/where changes depending on a number of things - primarily, those things are:

1. What is the "feature availability" for your specific resource?
2. Is your resource encrypted with a CMK or an AMK? This question is even more important if your resource does not support "Full Management".
3. Where do you want to backup to? Cross-account? Cross-region? To a Logically-Air-Gapped Vault? 

> **_NOTE:_**: CMK - Customer-Managed Key, AMK - Amazon Managed Key

> **_NOTE:_**: `"Full Management" == "Independent Encryption"`. If a resource supports Full Management, it means the resource supports Independent Encryption. This means that the backup is able to be immediately encrypted with a key .

The approach you take to implementing a solution with AWS Backup will very likely depend on what your end goal is. If you want to use AWS Backup in order to have backups that can be used to meet a tight RPO, where you're concerned about recovering from accidental corruption or deletion of data and getting back online as soon as possible, your backup plan/policy will likely differ from the use-case where you're concerned more about recovering from a situation where your AWS account is compromised, or where your database has suffered a ransomware attack. 

For example, for the first use-case, in-account backups may be completely acceptable, whereas in the second scenario, an account compromise could see your backups deleted and you left without anything to restore from. This blog focuses on implementing AWS Backup from the perspective of Disaster Recovery, e.g. in the event of account compromise or a ransomware attack.

## Backup Strategy

The first thing to understand/decide is this: how do you want to store and secure your backups? There are two primary avenues you can take here:

1. Store backups within the same account, storing backups in a LAGV (Logically Air-Gapped Vault) and utilising MPA (Multi-Party Approval)
2. Store backups in a dedicated DataBunker account - with this option, you can choose to use a LAGV and/or a regular Vault.

The reason that this choice matters is that it then informs how you need to go about implementing the solution. 

> **_NOTE:_**: There is also a question to be answered about cross-region backups, but I won't cover that just yet.

### Option 1: In-Account Backups + LAGV + MPA

A caveat straight off-the-bat - I describe these backups as "In-Account", but in reality, you'll be glad to hear the backups do technically live in an AWS-owned account - that is the nature of using a Logically Air-Gapped Vault (LAGV). However, from a logical view, the LAGV is _in_ the account, and the backups are visible from that view. 


### Option 2: DataBunker Backups

TBC


// TBC
## Resources that support "Full Management"

If the resources you want to backup all support **Full Management** - Congratulations! Your job just got a whole lot easier. Whilst "Full Management` does mean a number of things, from an implementation perspective, the thing you care about most is that it means it supports **Independent Encryption**. 

Independent Encryption means that the source resource (the one being backed-up) is able to be encrypted by AWS Backup with a key that is different to the key that the source resource is currently using. This is actually possible for resources that don't support "Full Management", but it requires 

## Resources that don't support "Full Management"

If some of the resources you want to backup **don't** support "Full Management" - Welcome to Hell. Well, it's not that bad, but it is annoying. 

### CMK-encrypted

If your source resource is **CMK**-encrypted, that's a good thing. 

Backing up to either a Vault in a separate account, or a Logically Air-Gapped Vault (LAGV), both require that the source resource is encrypted with a CMK. This is because a regular AMK cannot have a resource policy, and therefore you cannot update the resource policy to allow a cross-account action. 

With a CMK, you can update the resource policy to allow your secure Vault to decrypt the backup, enabling it to re-encrypt it with it's own KMS key.

### AMK-encrypted

If the source resources is **AWK**-encrypted, that's.. not ideal. You have three options:

1. If possible, re-encrypt the resource with a CMK.
2. 

## Resources that don't support LAGV




