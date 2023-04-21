---
title: "Cloud Build Trigger with Inline YAML via Terraform"
subtitle: "providing ad-hoc cloud build config via terraform"
date: 2023-04-21
author: rogerthat terraformer
tags: ["terraform", "cloud build"]
---

![banner](/img/inline-config.png)

When you configure a Cloud Build trigger via the Cloud Console, you have the
option of specifying the the `cloubuild.yaml` configuration as either a file
location on the repository or as an inline configuration.  I'm going to show you
how you can create a Cloud Build trigger with inline configuration using
Terraform. We're gonna make use of Terraform's [templatefile() function](https://developer.hashicorp.com/terraform/language/functions/templatefile).

Follow along with the [code available here](https://github.com/rogerthatdev/cloud-build-terraform/tree/main/inline-yaml).
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


## Use a yaml.tftpl file

You can acheive the above configuration using a Terraform template file that 
looks like the familiar cloudbuild.yaml file:

```
# 02-simple.cloudbuild.yaml.tftpl
steps:
  - name: ubuntu
    args:
      - echo
      - hello world hey
timeout: 600s

```

I can pass along values for the `step` block using a local value:

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
When deployed, the trigger will have the same in-line configuration as the 
example above:
```
steps:
  - name: ubuntu
    args:
      - echo
      - hello world hey
timeout: 600s

```
To pass along values for more than one step, I'll use a dynamic block.


## The dynamic `step` block

Here's a new template file with two steps:

```
# 03-multi-step-dynamic.cloudbuild.yaml.tftpl
steps:
  - name: ubuntu
    args:
      - echo
      - hello world hey
  - name: ubuntu
    args:
      - echo
      - peace out world
timeout: 600s

```
And here's our trigger configuration:

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
I used a `for_each` in a dynamic block to dynamically create a `step` block for 
each step in the template file. 

## Values for template variables 
There are a couple of ways to pass along values for variables you might want to 
use in a Terraform template file. The first one is by passing values through the 
`templatefile()` function:
```
locals {
  # You can pass values for variables in the yaml config via the second 
  # parameter of the templatefile() function:
  simple_config_04 = yamldecode(templatefile("${path.root}/cloudbuild/04-multi-step-dynamic-with-vars.cloudbuild.yaml", { "variable_here" = "WORLD", "another_variable" = "MARS" }))
}
```

Here's the template file for reference:

```
# 04-multi-step-dynamic-with-vars.cloudbuild.yaml.tftpl
steps:
  - name: ubuntu
    args:
      - echo
      - hello ${variable_here} hey
  - name: ubuntu
    args:
      - echo
      - peace out ${another_variable}
timeout: 600s

```

This results in the following Cloud Build config inline:

```
steps:
  - name: ubuntu
    args:
      - echo
      - hello WORLD hey
  - name: ubuntu
    args:
      - echo
      - peace out MARS
timeout: 600s
```
If you want to use a built-in variable in this configuration, like 
`$PROJECT_ID` or `$SHORT_SHA` you have escape the `$` character by adding an 
extra `$` (eg: `$$PROJECT_ID` AND `$SHORT_SHA`)

## Using the `substitutions` parameter

A better option for passing values to variables is the substitutions parameter. 
To do this, I have to escape the `$` character where I want variables:

```
# 05-multi-step-dynamic-with-subs.cloudbuild.yaml.tftpl
steps:
  - name: ubuntu
    args:
      - echo
      - hello $${variable_here} hey
  - name: ubuntu
    args:
      - echo
      - peace out $${another_variable}
timeout: 600s
```
Now when I bring in the template I don't need to supply any values via the 
`templatefile()` function:
```
locals {
  # Pass no values to the templatefile() function:
  simple_config_05 = yamldecode(templatefile("${path.root}/cloudbuild/05-multi-step-dynamic-with-subs.cloudbuild.yaml.tftpl", {}))
}
```
I instead add values in the substitutions parameter:
```
resource "google_cloudbuild_trigger" "simple_inline_05" {
  project     = var.project_id
  location    = var.region
  name        = "05-multi-step-dynamic-with-subs"
  description = "This trigger is deployed by terraform, with an in-line build config."
  trigger_template {
    branch_name  = "^main$"
    invert_regex = false
    project_id   = var.project_id
    repo_name    = google_sourcerepo_repository.placeholder.name
  }
  build {
    images        = []
    # This is where you'll supply the values for your variables
    substitutions = {
      variable_here = "WORLD"
      another_variable = "MARS"
    }
    tags          = []
    # This is the same dynamic block used in trigger 03 and 04 above
    dynamic "step" {
      for_each = local.simple_config_05.steps
      content {
        args = step.value.args
        name = step.value.name
      }
    }
  }
}
```
And with that, a trigger will be configured with the following in-line 
configuration:

```
steps:
  - name: ubuntu
    args:
      - echo
      - 'hello ${variable_here} hey'
  - name: ubuntu
    args:
      - echo
      - 'peace out ${another_variable}'
timeout: 600s
substitutions:
  another_variable: MARS
  variable_here: WORLD
```

