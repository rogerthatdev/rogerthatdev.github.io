---
title: "Cloud Build Trigger with Inline YAML via Terraform"
subtitle: "providing ad-hoc cloud build config via terraform"
date: 2023-04-12
author: rogerthat terraformer
tags: ["terraform", "cloud build"]
---

![banner](/img/inline-config.png)

When creating a Cloud Build trigger in the Cloud Console, you can specify an "in-line" location for your build configuration, essentially bundling the trigger with the config.  If you do this with Terraform, you'll be able to bundle the build config with the Terraform resource. Here's how.  
<!--more-->

## Simplest example

```
resource "google_cloudbuild_trigger" "simplest_inline" {
  project     = var.project_id
  location    = var.region
  name        = "01-simplest-inline-config-example"
  description = "This trigger is deployed by terraform, with an in-line build config."
  trigger_template {
    branch_name  = "^main$"
    invert_regex = false
    project_id   = var.project_id
    repo_name    = google_sourcerepo_repository.placeholder.name
  }
  build {
    images        = []
    substitutions = {}
    tags          = []
    # This is a single step defined inline
    step {
      name = "ubuntu"
      args = [
        "echo",
        "hello world hey"
      ]
    }
    timeout = "600s"
  }
}
```

## Use a yaml file

```
locals {
  # we bring in the yaml content via yamldecode() and templatefile()
  simple_config_02 = yamldecode(templatefile("${path.root}/cloudbuild/02-simple.cloudbuild.yaml", {}))
}

resource "google_cloudbuild_trigger" "simple_inline_02" {
  project     = var.project_id
  location    = var.region
  name        = "02-simple-inline-config-example"
  description = "This trigger is deployed by terraform, with an in-line build config."
  trigger_template {
    branch_name  = "^main$"
    invert_regex = false
    project_id   = var.project_id
    repo_name    = google_sourcerepo_repository.placeholder.name
  }
  build {
    images        = []
    substitutions = {}
    tags          = []
    # The single step gets values for name and args from the yaml brought in via
    # the local value above. This works because there's only one step.
    step {
      name = local.simple_config_02.steps[0].name
      args = local.simple_config_02.steps[0].args
    }
  }
}
```

## The dynamic `step` block

```
locals {
  # once again: we bring in the yaml content via yamldecode() and templatefile()
  simple_config_03 = yamldecode(templatefile("${path.root}/cloudbuild/03-multi-step-dynamic.cloudbuild.yaml", {}))
}

resource "google_cloudbuild_trigger" "simple_inline_03" {
  project     = var.project_id
  location    = var.region
  name        = "03-simple-inline-config-example"
  description = "This trigger is deployed by terraform, with an in-line build config."
  trigger_template {
    branch_name  = "^main$"
    invert_regex = false
    project_id   = var.project_id
    repo_name    = google_sourcerepo_repository.placeholder.name
  }
  build {
    images        = []
    substitutions = {}
    tags          = []
    # This is the dynamic block that will create a step block for each step in
    # the cloud build config file defined in the locals block above. This will
    # with as many steps as the build config file may have.
    dynamic "step" {
      for_each = local.simple_config_03.steps
      content {
        args = step.value.args
        name = step.value.name
      }
    }
  }
}
```

## Values for template variables 

```
locals {
  # You can pass values for variables in the yaml config via the second 
  # parameter of the templatefile() function:
  simple_config_04 = yamldecode(templatefile("${path.root}/cloudbuild/04-multi-step-dynamic-with-vars.cloudbuild.yaml", { "variable_here" = "WORLD", "another_variable" = "MARS" }))
}

resource "google_cloudbuild_trigger" "simple_inline_04" {
  project     = var.project_id
  location    = var.region
  name        = "04-simple-inline-config-example-with-vars"
  description = "This trigger is deployed by terraform, with an in-line build config."
  trigger_template {
    branch_name  = "^main$"
    invert_regex = false
    project_id   = var.project_id
    repo_name    = google_sourcerepo_repository.placeholder.name
  }
  build {
    images        = []
    substitutions = {}
    tags          = []
    # This is the same dynamic block used in trigger 03 above
    dynamic "step" {
      for_each = local.simple_config_04.steps
      content {
        args = step.value.args
        name = step.value.name
      }
    }
  }
}
```