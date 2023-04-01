---
title: "Terraform using Google Cloud Build"
subtitle: "A very basic example"
date: 2023-04-01
author: rogerthat terraformer
tags: ["terraform", "cicd", "cloud build"]
---
CI/CD can get pretty complicated. There are a lot of moving pieces and a lot of automation. But as they say, the only way to eat an elephant is one bite at a time. So here’s a first nibble.

## How basic are we talking here?

We’re going to use Cloud Build to run Terraform for us in our Google Cloud Project. And when I say a *very basic example*, I mean a **very** basic example. This won’t be anything you’ll use in production, but this will be the start of a series of posts that build towards a more intricate solution, discussing some important concepts and considerations along the way.

## My friend Terraform and my buddy Cloud Build

Those already familiar with Terraform already know the 3 magic commands we use to deploy resources to Google Cloud, or any other cloud provider out there: `terraform init` to initialize, followed by `terraform plan` to preview, and `terraform apply` to deploy. Manually running these commands is useful for learning Terraform or managing your own personal infrastructure in the Cloud, but in practice you’re going to need to learn how to automate Terraform. Let some robots do it for you. There plenty of ways to do that. Using Cloud Build is just one.

Cloud Build is Google Cloud’s serverless CI/CD platform. You give it a yaml configuration with some instructions and it can build, test and deploy your software. I won’t get into any more details than that — [the documentation](https://cloud.google.com/build/docs) does a much better job than I can.

## Where to begin?

First you’ll need a directory to work out of (a root Terraform module) with 2 files in it (for now): `cloudbuild.yaml` and `main.tf`. You can go and create these yourself, or just clone [this branch in this repository](https://github.com/rogerthatdev/cloud-build-terraform/tree/v1) that has the files already pre-written for you.

You’ll also need a Google Cloud Project with the [Cloud Build API](https://console.cloud.google.com/apis/library/cloudbuild.googleapis.com) enabled. When you enable the Cloud Build API, your project will generate a default Cloud Build service account that looks like this:

`98476438052@cloudbuild.gserviceaccount.com`

This service account automatically has the Cloud Build Service Account IAM role assigned to it, allowing it to do all the Cloud Build things it needs to be able to do, like manage Google Cloud Storage buckets and access Artifact Registry. You can see a full list of permissions [right here](https://cloud.google.com/build/docs/cloud-build-service-account#default_permissions_of_service_account).

## Tell Terraform what to do

Terraform gets its instructions from `.tf` files. For the sake of simplicity, let’s use this `main.tf` file I copied and pasted from one of [Hashicorp’s documentation pages](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/string):

```
# main.tf 

resource "random_string" "random" {
  length           = 16
  special          = true
  override_special = "/@£$"
}

output "random_string" {
    value = random_string.random.id
}
```

There’s nothing fancy to see here. It just creates a random string resource and spits its value out as output.

## Tell Cloud Build what to do

Cloud Build accepts yaml configuration files. Here is a simple one that will execute those 3 magic Terraform commands:

```
# cloudbuild.yaml

steps:
  - id: 'terraform init'
    name: 'hashicorp/terraform:1.0.0'
    script: terraform init
  - id: 'terraform plan'
    name: 'hashicorp/terraform:1.0.0'
    script: terraform plan
  - id: 'terraform apply'
    name: 'hashicorp/terraform:1.0.0'
    script: terraform apply --auto-approve
```

The id field is the name that will show up in the logs, and the name field is the public image that it will build and use to execute the provided command.

You can kick off a Cloud Build job by running `gcloud builds submit .` in your working directory. It will automatically look for a `cloudbuild.yaml` file and create a Cloud Build job that will execute the provided instructions. You can see the progress as output in your terminal or head over the [Cloud Build History page](https://console.cloud.google.com/cloud-build/builds) where you’ll see your job’s status and output:

![screenshot of build details](/img/builddetails.png)

And there you have it! Cloud Build executing Terraform! That’s all there is to it!

## Wait a second, there must be more to it

Okay, actually there’s a lot more to it. While we did indeed run Terraform with Cloud Build we didn’t actually deploy any infrastructure (or anything useful for that matter). There are some things that we need to consider.

## Permissions

Cloud Build creates a default service account with [those default Cloud Build service account permissions](https://cloud.google.com/build/docs/cloud-build-service-account#default_permissions_of_service_account) assigned to it. Any additional permissions that this service accounts needs will depend on the cloud resources that you want your Terraform to create. If you’re managing resources across lots of different Google Cloud services, then the Cloud Build service account will inevitably accumulate excessive permissions, resulting in a potential security risk. Yuck.

## Terraform state file

You’ll notice that the Terraform state file is no where to be found. The state file that would have been generated to your local disk had you ran `terraform` apply locally only existed temporarily in the container that Cloud Build ran the command from. If this Terraform had created a cloud resource, you would have no way to update its parameters using a second `terraform apply` or destroy it using `terraform destroy`. Ugh.

## That `--auto-approve` flag

If this were an automated pipeline, we probably wouldn’t be running `apply` automatically after `plan`. Without approvals, reviews, or staging environments in place, we can’t really make sure our infrastructure doesn’t fall apart after each terraform job. Yikes.

## Key takeaways

- Enabling the Cloud Build API will create a default Cloud Build service account in your project. Cloud Build jobs run using this service account by default.
- Cloud Build’s instructions are provided as steps listed in a yaml file, each one with it’s own output that show up in the build logs. If the Terraform creates any cloud resources, the service account will need the appropriate permissions to manage those resources.
- Without any remote state configured, the state file will only exist temporarily and then get lost in the abyss.

## Well, what next?

While limited in its utility, this example can serve as a starting point for your own exploration around the considerations I mentioned above. Service account impersonation will definitely play a role in ironing out the permissions (here’s a blog about Terraform and service account impersonation I published [here](https://cloud.google.com/blog/topics/developers-practitioners/using-google-cloud-service-account-impersonation-your-terraform-code)), while Terraform’s support for [remote state files on Google Cloud Storage](https://developer.hashicorp.com/terraform/language/settings/backends/gcs) will give us the persistent state that we need to keep our infrastructure updated safely. As for the lack of approvals and reviews, there are unlimited ways to design your pipeline to accommodate such requirements with features like [gating builds on approvals](https://cloud.google.com/build/docs/securing-builds/gate-builds-on-approval). I won’t leave you high and dry though. I will post a follow up soon with another bite of our elephant for us to explore.