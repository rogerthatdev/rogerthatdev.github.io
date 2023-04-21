---
title: "Cloud Build Trigger with Inline YAML via Terraform"
subtitle: "providing ad-hoc cloud build config via terraform"
date: 2023-04-12
author: rogerthat terraformer
tags: ["terraform", "cloud build"]
---

![banner](/img/inline-config.png)

When you configure a Cloud Build trigger via the Cloud Console, you have the
option of specifying the the `cloubuild.yaml` configuration as either a file
location on the repository or as an inline configuration.  I'm going to show you
how you can create a Cloud Build trigger with inline configuration using
Terraform. We're gonna make use of Terraform's [templatefile() function](https://developer.hashicorp.com/terraform/language/functions/templatefile).    
<!--more-->

## Simplest example
Here's a basic Cloud Build Trigger configured with a Cloud Source Repo I made 
somewhere else in this `.tf` file:

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
    # I can't create a trigger without connecting a repo to the project, so I 
    # made a Cloud Source Repo and passed it along here
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

It's a single step, passed along to the `build` block. This trigger will have the following Cloud Build config 
configured inline:

```yaml
steps:
  - name: ubuntu
    args:
      - echo
      - hello world hey
timeout: 600s

```
This works great when it's one step, but could be impractical for configs that 
have multiple steps and updated often.  This is where `tftpl` files come in 
handy.

## Use a yaml.tftpl file
By using a Terraform template file that looks like the familiar cloudbuild.yaml 
file, you can easily configure a trigger with multiple steps, configured inline:

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