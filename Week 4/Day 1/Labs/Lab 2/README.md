# Azure Blob Storage Lab - CLI

## What I Did

Used Azure CLI on Ubuntu to provision a Blob Storage account, create a container,
upload a file, download it back, and generate a temporary SAS URL for sharing.
Cleaned up all resources at the end by deleting the resource group.

---

## Problem I Hit

When I got to the upload step, I kept getting a permissions error telling me I needed
the Storage Blob Data Contributor role. The lab uses --auth-mode login which goes
through Azure AD and checks your role assignments. In my subscription I did not have
permission to assign that role to myself, so every blob operation was getting denied.

I fixed it by fetching the storage account key and using --account-key on all the
storage commands instead. That bypasses the role system and authenticates directly
with the account key.

    ACCOUNT_KEY=$(az storage account keys list \
      --account-name $SA_NAME \
      --resource-group $RESOURCE_GROUP \
      --query "[0].value" -o tsv)

After that everything worked fine.

---

## What I Learned

- How to create and manage Azure Blob Storage resources entirely from the terminal
- The difference between --auth-mode login (Azure AD / RBAC) and --account-key auth
- What a SAS URL is and how it gives temporary read access to a file without
  requiring the other person to have an Azure account
- How deleting a resource group removes everything inside it at once, which is
  the clean way to tear down a lab environment
