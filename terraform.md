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