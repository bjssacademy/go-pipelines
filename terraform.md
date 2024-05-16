# Terraform

Terraform is an open-source *infrastructure as code* (IaC) tool created by HashiCorp that allows users to define and provision infrastructure resources using a declarative configuration language, HashiCorp Configuration Language (HCL), to describe the desired state of infrastructure resources.

## IaC

Infrastructure as Code (IaC) is an approach to managing and provisioning infrastructure resources using *machine-readable configuration files* rather than manually configuring hardware or using interactive configuration tools (aka "ClickOps"). With IaC, infrastructure components such as virtual machines, networks, storage, and middleware are defined and managed using code.

## Highlights

Terraform is primarily used for provisioning and managing infrastructure resources across various cloud providers, including AWS, Azure, Google Cloud Platform (GCP), and others.

It enables users to provision infrastructure across *multiple* cloud providers and on-premises environments, providing flexibility and avoiding vendor lock-in. Unlike the YAML we have just created, terraform can be used on any cloud provider - however, each cloud provider has different names for resources, so it's not "write once use anywhere"!

## Example

Okay, here's an example of terraform, based on our previous YAML file:

```tf
provider "azurerm" {
  features {}
}

resource "azurerm_devops_serviceendpoint_azurerm" "acr" {
  name                = "acr_service_connection"
  azure_rm_service_id = "YOURAZURERM_SERVICE_PRINCIPAL_ID"
  azure_rm_tenant_id  = "YOURAZURERM_TENANT_ID"
  subscription_id     = "YOURAZURERM_SUBSCRIPTION_ID"
}

resource "azurerm_devops_pipeline" "example_pipeline" {
  project_id = "YOURAZURERM_PROJECT_ID"
  name       = "example_pipeline"
  folder_path = "/"

  repository {
    repo_type = "azureReposGit"
    repo_name = "YOURAZURE_REPO_NAME"
    branch_name = "main"
  }

  trigger {
    branches = ["main"]
  }

  variables {
    docker_registry_service_connection = "YOURACRNAME"
    image_repository                   = "YOURIMAGENAME"
    container_registry                 = "YOURACRNAME.azurecr.io"
    dockerfile_path                    = "\$(Build.SourcesDirectory)/Dockerfile"
    tag                                = "\$(Build.BuildId)"
    azure_subscription                 = "YOURSERVICECONNECTOR"
    app_name                           = "YOURWEBAPPNAME"
  }

  agent_pool {
    name = "ubuntu-latest"
  }

  stage {
    name = "Build"
    jobs {
      job {
        name   = "Build"
        steps {
          task {
            task_type       = "Docker@2"
            display_name    = "Build"
            inputs = {
              command             = "buildAndPush"
              repository          = "\${{ variables.image_repository }}"
              dockerfile_path     = "\${{ variables.dockerfile_path }}"
              container_registry  = "\${{ variables.docker_registry_service_connection }}"
              tags                = "\${{ variables.tag }}"
            }
          }
        }
      }
    }
  }

  stage {
    name = "Test"
    depends_on = ["Build"]
    jobs {
      job {
        name   = "Test"
        steps {
          script {
            code = <<-EOT
              echo "Running tests..."
              # Run the Go tests from the project directory
              cd /app
              go test ./...
            EOT
          }
        }
      }
    }
  }

  stage {
    name = "Deploy"
    depends_on = ["Test"]
    jobs {
      job {
        name   = "Deploy"
        steps {
          task {
            task_type    = "AzureWebAppContainer@1"
            display_name = "Deploy Azure Web App"
            inputs = {
              azure_subscription = "\${{ variables.azure_subscription }}"
              app_name           = "\${{ variables.app_name }}"
              containers         = "\${{ variables.container_registry }}/\${{ variables.image_repository }}:\${{ variables.tag }}"
            }
          }
        }
      }
    }
  }
}

```

- We're using the `azurerm_devops_serviceendpoint_azurerm `resource to create an Azure DevOps service connection to Azure Container Registry (ACR). You need to replace the placeholders (e.g., YOURAZURERM_*) with your actual values.
- We define pipeline variables using the `variables` block.
-The `agent_pool` block specifies the VM image to use for pipeline execution.
- Each *stage* and *job* is defined within the corresponding blocks.
- Tasks within jobs are defined using the appropriate task types (task or script).
- Placeholder values (e.g., YOURACRNAME, YOURIMAGENAME, etc.) should be replaced with your actual values!

---

## More Terraform

[Here's the link to the terraform getting started guide on Azure](https://developer.hashicorp.com/terraform/tutorials/azure-get-started/azure-build)

## Install

In WSL:
```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

### Login to Azure from the commandline

```bash
az login --tenant 1491bbb5-26ff-4146-a689-da9d7f9df86f
```

Follow the instructions to log into your BJSS Azure account in a browser

Official Install docs: https://docs.microsoft.com/en-us/cli/azure/install-azure-cli


### Install Terraform

* Install the Binaries

```bash
 wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
 echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
 sudo apt update && sudo apt install terraform
```

* Verify the Install
```bash
terraform -version
```

Official Install Docs:
https://learn.hashicorp.com/tutorials/terraform/install-cli

## Create Terraform Infrastructure with Docker

In the terraform-docker folder, open the `terraform.tf` file.

This file includes the terraform block, which defines the provider and Terraform versions you will use with this project.

```tf
terraform {
  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = "~> 3.0.2"
    }
  }
  required_version = "~> 1.7"
}
```
Next, open `main.tf` and copy and paste the following configuration.

```tf
provider "docker" {}

resource "docker_image" "nginx" {
  name         = "nginx:latest"
  keep_locally = false
}

resource "docker_container" "nginx" {
  image = docker_image.nginx.image_id
  name  = "tutorial"
  ports {
    internal = 80
    external = 8000
  }
}
```

### Terraform init

When we start working with our Terraform project we need to first initialise it.

*"The terraform init command is used to initialize a working directory containing Terraform configuration files. This is the first command that should be run after writing a new Terraform configuration or cloning an existing one from version control. It is safe to run this command multiple times."*

In the terminal, initialize the project, which downloads a plugin that allows Terraform to interact with Docker.

```bash
terraform init
```
### Terraform plan
The terraform plan command is used to create an execution plan. Terraform performs a refresh, and determines what actions it needs to take to achieve the desired state specified in the configuration files.
This command is a convenient way to check whether the execution plan for a set of changes matches your expectations without making any changes to real resources or to the state. For example, terraform plan might be run before committing a change to version control, to create confidence that it will behave as expected.

Running a Plan:

```bash
terraform plan
```

### Terraform apply

The terraform apply command is used to apply the changes required to reach the desired state of the configuration.

Provision the NGINX server container with apply. When Terraform asks you to confirm, type yes and press ENTER.

```bash
terraform apply
```

### Verify NGINX instance

Run `docker ps` to view the NGINX container running in Docker via Terraform.

```bash
docker ps
```

### Destroy resources
To stop the container and destroy the resources created in this tutorial, run `terraform destroy`. When Terraform asks you to confirm, type yes and press ENTER.

```bash
terraform destroy
```

You have now provisioned and destroyed an NGINX webserver with Terraform.