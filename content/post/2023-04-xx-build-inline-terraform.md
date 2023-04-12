---
title: "Cloud Build Trigger with Inline YAML via Terraform"
subtitle: "providing ad-hoc cloud build config via terraform"
date: 2023-04-12
author: rogerthat terraformer
tags: ["terraform", "cloud build"]
---

![banner](/img/buildform.png)

When creating a Cloud Build trigger in the Cloud Console, you can specify an "in-line" location for your build configuration, essentially bundling the trigger with the config.  If you do this with Terraform, you'll be able to bundle the build config with the Terraform resource. Here's how.  
<!--more-->

