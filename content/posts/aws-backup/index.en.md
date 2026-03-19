---
weight: 2
title: "AWS Backup -> A Primer"
date: 2025-12-07T00:00:00+00:00
lastmod: 2025-12-07T00:00:00+00:00
draft: false
author: "Jack"
authorLink: "https://jackroberts.uk"
description: "Understanding the different challenges when using AWS Backup"
images: []

tags: ["aws", "backup", "ransomware", "data", "recovery", "cloud", "disaster recovery"]
categories: ["post"]

lightgallery: true

toc:
  auto: false
---

## Introduction

AWS Backup is a great solution for backing up supported AWS resources, allowing you to use AWS-native tools and resources to safeguard your stateful, supported AWS resources. 

You'll notice above, I use the word "supported" a number of times. That is because AWS Backup does not support backup for _all_ stateful resources - at least not at the time of writing.

The good news is that AWS Backup does support most of the "big ones", as you should hopefully see from the table I link in the next paragraph. There is a catch, however, and it's this: amongst the resources that are supported, there is a difference in "feature availability" from one resource type to the next. For example, what is supported for a DynamoDB Table versus an EBS Volume.

See here for AWS Backup Feature Availability: https://docs.aws.amazon.com/aws-backup/latest/devguide/backup-feature-availability.html 

Now, what does this mean in practise? Ultimately you **will** be able to backup your resources as long as they're supported, but the how/what/where changes depending on a number of things - primarily, those things are:

1. What is the "feature availability" for your specific resource?
2. Is your resource encrypted with a CMK or an AMK? This question is even more important if your resource does not support "Full Management" (see backup feature availability table)
3. Where do you want to backup to? Cross-account? Cross-region? To a Logically-Air-Gapped Vault? 

> **_NOTE:_**: CMK = Customer-Managed Key, AMK = Amazon Managed Key, AOK = Amazon Owned Key

> **_NOTE:_**: `"Full Management" == "Independent Encryption"`. If a resource supports Full Management, it means the resource supports Independent Encryption. This means that the backup is able to be immediately encrypted with a key that is different to what the source resource uses.

The approach you take to implementing a solution with AWS Backup will very likely depend on what your end goal is. If you want to use AWS Backup in order to have backups that can be used to meet a tight RPO, where you're concerned about recovering from accidental corruption or deletion of data and getting back online as soon as possible, your backup strategy will likely differ from the use-case where you're concerned more about recovering from a situation where your AWS account is completely compromised, or where your database has suffered a ransomware attack. 

For example, for the first use-case, in-account backups may be completely acceptable, whereas in the second scenario, an account compromise could see your backups deleted and you left without anything to restore from. This blog focuses on implementing AWS Backup from the perspective of Disaster Recovery, e.g. in the event of account compromise or a ransomware attack.

## Backup Strategy

### Policy vs Plans

Quick primer - a Backup Plan is a configuration that says what resources should be backed up, when, and to where. It orchestrates the backups. You can create a Backup Plan directly, or have them be created automatically via a Backup Policy if using AWS Organizations.

The first decision is a simple one - should you use a Backup Policy, or distribute Backup Plans (e.g. via a StackSet or deployed separately)?

If you are using AWS Organizations, go with a policy. They ultimately deploy plans within the target accounts, which are targetted via OU, but they also come with built-in protection/immutability so they cannot be deleted by a principal within the target account, saving you from creating an SCP to handle that.

If you aren't using AWS Organizations, you're stuck with Backup Plans. In this case, I'd recommend distributing to target accounts via e.g. a StackSet, and protecting them from deletion via an SCP.

Easy - the next decision is a little trickier:

## In-Account vs Out-of-Account Backups

There are really two options you should consider:

1. Store backups in an AWS Logically Air-Gapped Vault (LAGV) - you can do this in-account, because the backups are __actually__ stored in an AWS-managed account transparently. 

2. Store backups in a dedicated, locked-down DataBunker account - with this option, you can choose to use a LAGV or a regular Vault with compliance mode enabled.

The reason that this choice matters upfront is that it informs how you need to go about implementing your AWS Backup solution. I'll expand on this later.

> **_NOTE:_**: There are also implications of performing cross-region backups, but I won't cover that just yet.

### Option 1: In-Account Backups with LAGV + MPA (Multi-Party Approval)

A caveat straight off-the-bat - I describe these backups as "In-Account", but in reality, as mentioned briefly above, you'll be glad to hear the backups do technically live in an AWS-owned account - that is the nature of using a Logically Air-Gapped Vault (LAGV). However, from a logical view, the LAGV is _in_ the account, and the backups are visible there.

In this option, you deploy a LAGV in the same account as where your resource(s) that you're backing up are deployed. In this solution, you must also utilise MPA (Multi-Party Approval), another AWS Service, as the mechanism by which you would be able to recover backups from a LAGV that may live within a compromised/inaccessible account. You do this by setting up MPA, configuring a team, and then attaching that team to the LAGV. 

> **_NOTE:_**: Assigning a team to a LAGV is a one-time operation. Once it's done, you can neither change nor remove the assignment.

Using this pattern, you end up with a pattern where each workload has it's own Vault, reducing the blast radius from "one central vault" to several dedicated vaults. There are no extra costs to having more vaults - you only pay for the storage you use.


Finally, the Backup Plan deployed in the account is then responsible for having the correct configuration to backup the resource to the LAGV. This isn't as simple as it sounds - the configuration you'll need will vary based on resource type, and encryption type. We'll go in to this later.

### Option 2: DataBunker Backups

In this option, the typical pattern is to create a DataBunker account in which you deploy a central Backup Vault. You the configure Backup Plan's to perform cross-account copies of your backups to this central vault.

Ideally, you should create this account within a Business Continuity OU with locked down controls via SCP's, and ideally the principal's who can access this account should not be your engineers/the same people who have access to the primary copy of the data (i.e. the running database/workload).

Finally, as with the previous option, the Backup Plan deployed in the account is then responsible for having the correct configuration to copy the backup to the central vault. Again, this isn't necessarily as simple as it sounds - the configuration you'll need will vary based on resource type, and encryption type.

## Encryption implications

### Resources that support "Full Management"

If the resources you want to backup all support **Full Management** - Congratulations! Your job just got a whole lot easier. Whilst "Full Management` does mean a number of things, from an implementation perspective, the thing you care about most is that it means the resource supports **Independent Encryption**. 

Independent Encryption means that you can create a direct backup of a resource with a KMS key that is different to the one the source resource (the one being backed-up)is currently using.

### Resources that don't support "Full Management"

If some of the resources you want to backup **don't** support "Full Management" - Welcome to Hell. Well, it's not that bad, but it is annoying, and how annoying depends on the following:

#### CMK-encrypted

If your source resource is **CMK**-encrypted, that's a good thing. With a CMK, you can update the resource policy to allow your secure Vault to decrypt the backup, enabling it to re-encrypt it with it's own KMS key and ultimately store the backup.

Why is a CMK required? Whether it's to a DataBunker vault or an in-account Logically Air-Gapped Vault (LAGV), both require that the source resource is encrypted with a CMK. This is because a regular AMK doesn't have a resource policy, and therefore you cannot update the resource policy to allow the cross-account action - which a copy to a DataBunker **is**, and which a copy/backup to a LAGV is **considered to be**. 

#### AMK-encrypted

If the source resources is **AWK**-encrypted, that's... not ideal. You have three options:

1. If possible, re-encrypt the resource with a CMK.
2. Have your backup plan perform a copy to an intermediate vault that is encrypted with a CMK - this will cause your backup to be re-encrypted with a CMK, within the account. You can then have a separate process outside of your Backup Plan (e.g. event -> lambda) to perform the copy to the DataBunker/LAGV.
3. Cry, give-up, go home.

### Resources that don't support LAGV

Some resources don't support Logically Air-Gapped Vaults. The good news is: it's reasonably rare, and support for more resource types is forthcoming from AWS. 

However, if your resource type doesn't support LAGV, you're likely stuck with the DataBunker approach, or some other non-AWS-Backup solution for those specific resource types.

## Primary Backup + CMK-encrypted LAG Vaults

A closing note on a couple of new features that have been released reasonably recently:

1. Primary Backups to LAG Vaults
2. CMK-encryption for LAG Vaults

Let's dive in to both of them:

### Primary Backup to LAGV

When a Backup Plan executes, it needs to perform an initial backup to a vault within the account first - this is required by AWS Backup. From there, you then configure a copy action in the Plan to make a secondary copy of the backup to another Vault. 

This **was** the case when backing up resources to a LAGV... until the release of "Primary Backup to LAGV". 

AWS recognised that there are situations where customers just want a singular, super-safe, air-gapped copy of their backup in the event of a disaster, and nothing more. They didn't want or need an intermediate copy - it adds overhead, and costs the customer unnecessarily.

With "Primary Backup to LAGV", you can configure your Backup Plan so that it directly backs up a resource to a LAGV, withou the need for an intermediate copy. This is supported for resources that have support for "Full Management", or where the resource does not support "Full Management" but is CMK encrypted, with the caveat that in the second scenario, AWS do technically create a temporary backup in a local in-account vault, which they then clean up. 

For more information, read here: https://docs.aws.amazon.com/aws-backup/latest/devguide/lag-vault-primary-backup.html

### CMK-encrypted LAGV

Historically, LAG Vaults were encrypted with an AOK - a key that Amazon own and control, so that you don't have to. For a lot of businesses, this is perfectly fine, and indeed recommended.

However, if your business has certain compliance requirements, it may be a requirement that your backups are encrypted with a key that is under your explicit control. For this requirement, you can choose to encrypt your LAG Vault with a CMK. 

You can read more here: https://aws.amazon.com/blogs/storage/encrypt-aws-backup-logically-air-gapped-vaults-with-customer-managed-keys/

It is generally advised that if you require this, you should create a separate "Key Material" AWS account which holds the KMS key, and adds controls that lockdown and monitor uses of, and actions on, that key material.

## Conclusion

It's a lot to take in - I'd recommend reading the following links to get started:

* https://docs.aws.amazon.com/aws-backup/latest/devguide/whatisbackup.html
* https://d1.awsstatic.com/architecture-diagrams/ArchitectureDiagrams/data-protection-with-aws-backup-ra.pdf?stod_bck4