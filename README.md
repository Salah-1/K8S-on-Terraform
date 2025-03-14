# Terraform constraints to prevent mis-configuration or accidental deletion.
The question came up that “can we have storage volume that doesn’t get deleted when the node or pod get recycled”? And my answer was absolutely, just use **constraints** in the **Terraform config**. After digging, the requirement boiled down to:

    1. Storage volume doesn’t get deleted upon deletion of pods or nods.
    2. All resources be created in the “US West” region.
    3. Make those changes so they can’t be overridden by kubectl. 
    
That translates to: 

Terraform configuration that ensures:

✅ All resources are created in the US West region

✅ Persistent Volumes (PVs) use the "standard" storage class

✅ Reclaim policy is set to "Retain"

✅ Resources cannot be overridden by kubectl


The infrastructure as Code (Iac) translation of the above using Terraform .tv is

Terraform Configuration (main.tf)
```
provider "kubernetes" {
  config_path = "~/.kube/config"
}

provider "google" {
  project = var.project_id
  region  = "us-west1"  # Ensuring resources are in US West region
}

# Enforce constraints using Terraform variables
variable "project_id" {
  description = "GCP Project ID"
  type        = string
}

variable "region" {
  description = "Enforced region for all resources"
  type        = string
  default     = "us-west1"
}

# Kubernetes Storage Class (standard, cannot be overridden)
resource "kubernetes_storage_class" "standard_storage" {
  metadata {
    name = "standard"
  }

  storage_provisioner = "kubernetes.io/gce-pd"

  parameters = {
    type = "pd-standard"
  }

  reclaim_policy      = "Retain"
  allow_volume_expansion = true

  lifecycle {
    prevent_destroy = true  # Ensures this storage class can't be deleted
  }
}

# Kubernetes Persistent Volume
resource "kubernetes_persistent_volume" "example_pv" {
  metadata {
    name = "example-pv"
  }

  spec {
    capacity = {
      storage = "100Gi"
    }

    access_modes = ["ReadWriteOnce"]

    persistent_volume_source {
      gce_persistent_disk {
        pd_name = "example-disk"
        fs_type = "ext4"
      }
    }

    storage_class_name          = kubernetes_storage_class.standard_storage.metadata[0].name
    persistent_volume_reclaim_policy = "Retain"
  }

  lifecycle {
    prevent_destroy = true  # Prevent accidental deletion
  }
}

# Kubernetes Persistent Volume Claim
resource "kubernetes_persistent_volume_claim" "example_pvc" {
  metadata {
    name = "example-pvc"
  }

  spec {
    access_modes = ["ReadWriteOnce"]

    resources {
      requests = {
        storage = "100Gi"
      }
    }

    storage_class_name = kubernetes_storage_class.standard_storage.metadata[0].name
  }
}
```
 How Does This Prevent kubectl Changes?
 
    1. Storage Class Cannot Be Overridden
    
        ◦ The standard storage class is declared in Terraform and marked with: 
          lifecycle {
            prevent_destroy = true
          }
          
        ◦ Prevents accidental deletion or modification. 
        
    2. Persistent Volume Uses Retain Policy
    
        ◦ This ensures data is not deleted if a PV is deleted: 
          persistent_volume_reclaim_policy = "Retain"
          
    3. Forces Use of US-West Region
    
        ◦ All resources are tied to "us-west1" to enforce regional placement. 
        
    4. Terraform Manages Everything
        ◦ If someone tries to manually modify these resources via kubectl, Terraform will detect drift and restore the correct configuration. 

**It's important to note that this guards against misconfig or accidental change of values and not malicious**.
