---
date: "2024-11-09T13:58:33+01:00"
draft: false
title: "Deleting a Databricks Workspace's Default SQL Warehouse with Terraform"
tags: ["databricks", "terraform"]

ShareButtons: [linkedin, reddit, x, whatsapp]
showtoc: true
---

## What is Databricks' Default SQL Warehouse?

An SQL warehouse in Databricks is a compute resource specifically tailored for handling SQL workloads and queries.
It's designed to support data analysis and BI (Business Intelligence) activities by enabling fast, efficient access to data stored in Databricks' Delta Lake and other data sources.

When you create a new Databricks workspace, it comes with a default warehouse.
The default warehouse is automatically created and cannot be bypassed.
However, you can remove the default warehouse after it's been created, either through the Databricks UI, the Databricks CLI, or the Databricks REST API.

## Why Delete the Default SQL Warehouse?

There are several reasons why you might want to delete the default SQL warehouse in Databricks:

1. **Cost Management**: The default warehouse (if it's not serverless) is billed based on the VM instance type and the time it's running. If you're not using the default warehouse, you can save costs by deleting it.
2. **Customization**: If you prefer to create your own SQL warehouse with specific configurations, you might want to delete the default warehouse and create a new one with the desired settings.
3. **Security**: By deleting the default warehouse, you can ensure that no one accidentally uses it for running queries or accessing data.

In my case, I wanted to delete the default serverless SQL warehouse and create a new non-serverless one with a specific configuration.

## Deleting the Default SQL Warehouse with Terraform

Since I manage the Databricks infrastructure with Terraform and the number of Databricks workspaces is growing,
I wanted to automate the deletion of the default SQL warehouse using Terraform. This way, I can ensure that the default warehouse is removed consistently across all workspaces.

In general, Terraform is used to create infrastructure resources by running the `terraform apply` command.
Although resources can also be deleted with Terraform by running the `terraform destroy` command,
it is not helpful in this case because I want the default warehouse deleted during the workspace creation process.

Nevertheless, the `local-exec` provisioner in Terraform can be used to execute shell commands on the local machine where Terraform is running.
I used this provisioner to delete the default SQL warehouse with a simple `curl` command to the Databricks REST API.

Here's the Terraform code snippet that deletes the default SQL warehouse:

```hcl
resource "terraform_data" "delete_default_warehouse" {
  triggers_replace = data.azurerm_databricks_workspace.this.id

provisioner "local-exec" {
    interpreter = ["/bin/bash", "-c"]
    command     = <<EOT
    set -eo pipefail
    WAREHOUSE_ID=$(curl --silent --request GET https://$WORKSPACE_URL/api/2.0/sql/warehouses \
      --header "Authorization: Bearer $DATABRICKS_TOKEN" | \
      jq -r --arg name "$WAREHOUSE_NAME" '.warehouses[] | select(.name == $name) | .id')

    if [ -n "$WAREHOUSE_ID" ]; then
      echo "Deleting default serverless SQL warehouse with ID: $WAREHOUSE_ID"
      curl --silent --request DELETE https://$WORKSPACE_URL/api/2.0/sql/warehouses/$WAREHOUSE_ID \
        --header "Authorization: Bearer $DATABRICKS_TOKEN"
    else
      echo "No warehouse named '$WAREHOUSE_NAME' found."
    fi
    EOT
    environment = {
      WORKSPACE_URL    = data.azurerm_databricks_workspace.this.workspace_url
      # We want to see the logs, so we don't mark this as sensitive (token is anyway not logged)
      DATABRICKS_TOKEN = nonsensitive(databricks_token.pat.token_value)
      WAREHOUSE_NAME   = "Serverless Starter Warehouse"
    }
  }
}

resource "databricks_token" "pat" {
  comment          = "Terraform Provisioning"
  lifetime_seconds = 60
}

data "azurerm_databricks_workspace" "this" {
  name                = "my-databricks-workspace"
  resource_group_name = "my-resource-group"
}
```

A few points should be noted about the Terraform code snippet:

- The creation of the Databricks workspace happens in a separate Terraform module because you need the workspace URL to configure the Databricks provider.
- The workspace information is fetched with the `data.azurerm_databricks_workspace` data source.
- The default SQL warehouse information is also fetched with a `curl` command to the Databricks REST API.
  The `jq` command is used to parse the JSON response and extract the warehouse ID.
- The `databricks_token` resource is used to create a Personal Access Token (PAT) for authenticating with the Databricks REST API.
  Its lifetime is set to 60 seconds to ensure that the token is short-lived.
- The `local-exec` provisioner (or any other provisioner) can be added to any resource in Terraform.
  In this case, it's added to a `terraform_data` resource which is a dummy resource that doesn't create any infrastructure.
- The `trigger_replace` attribute is used to force the `local-exec` provisioner to run every time the workspace ID changes.
  This ensures that the default warehouse is deleted whenever the workspace is recreated.
- A network connection between the machine running Terraform and the Databricks workspace is required to execute the `curl` command.

## Read More

The documentation for the Databricks REST API can be found at [Databricks REST API reference](https://docs.databricks.com/api/workspace/introduction)
and specifically for the `DELETE` method of an SQL warehouse: [Delete a warehouse](https://docs.databricks.com/api/workspace/warehouses/delete)

It's also worth reading the Terraform documentation on the `terraform_data` resource [The terraform_data Managed Resource Type](https://developer.hashicorp.com/terraform/language/resources/terraform-data)
as well as the `local-exec` provisioner [local-exec Provisioner](https://developer.hashicorp.com/terraform/language/resources/provisioners/local-exec).
