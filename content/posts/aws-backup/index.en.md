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

tags: ["go", "golang", "goroutines", "chan", "channels", "concurrency"]
categories: ["post"]

lightgallery: true

toc:
  auto: false
---

## Introduction

AWS Backup is a great solution for backing up AWS resources, allowing you to use AWS native tools and resources to back up your other AWS resources. However, it isn't all sunshine and roses.

The "catch" with AWS Backup is that the way in which you have to backup a resource, and where you can backup to, varies depending on the resource type, as well as by the type of KMS key the **source** resource is encrypted with.

What does this mean ultimately? Well, if you have lots of different resource types you want to backup, and/or if you have lots of teams doing encryption in different ways.. you're in for a fun time. As a Platform team, you'll need to construct policies that work for all the different use-cases, without exposing that complexity to your teams.

See below for AWS Backup Feature Availability:
https://docs.aws.amazon.com/aws-backup/latest/devguide/backup-feature-availability.html

> **_NOTE:_**: `"Full Management" == "Independent Encryption"`. If a resource supports Full Management, it means the resource supports Independent Encryption. This means that the backup is able to be immediately encrypted with a key that is different to the key that the source resource uses.

## Resources that support "Full Management"

If the resources you want to backup all support "Full Management" - Congratulations! Your job just got a whole lot easier.

## Resources that don't support "Full Management"

If some of the resources you want to backup **don't** support "Full Management" - Welcome to the trenches.

### CMK-encrypted

If your source resource is **CMK**-encrypted, that's a good thing. 

Backing up to either a Vault in a separate account, or a Logically Air-Gapped Vault (LAGV), both require that the source resource is encrypted with a CMK. This is because a regular AMK cannot have a resource policy, and therefore you cannot update the resource policy to allow a cross-account action. 

With a CMK, you can update the resource policy to allow your secure Vault to decrypt the backup, enabling it to re-encrypt it with it's own KMS key.

### AMK-encrypted

If the source resources is **AWK**-encrypted, that's.. not ideal. You have three options:

1. If possible, re-encrypt the resource with a CMK.
2. 

## Resources that don't support LAGV




