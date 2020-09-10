# Private GKE in Shared VPC

The purpose of this code base is to create a Shared VPC, with 2 subnets and 2 GKE clusters in each subnet.  We are going to use a custom service account for the node pools, so that we don't use the default Compute service account, which is overprivileged.

We are not going to use any Terraform modules, as it's difficult to understand what is happening in this approach.  This guide will walk you through all the components that are required to replicate the architecture described in the following section.

There is already a [great article](https://cloud.google.com/kubernetes-engine/docs/how-to/cluster-shared-vpc) out there that describes all the individual steps in more detail.  However, it uses `gcloud`-commands and it doesn't elaborate on some steps as much as I would like to. 

## Components
![Architecture](./architecture)

The following components will be created as part of this article:
- Host project
- Service project (2)
- Network and subnets (2)
- Custom service account (2) for the node pools (2)
- GKE cluster (2)

## Service Account
Normally, the GKE cluster runs with the default Compute service account.  Because this service account has very open permissions (Editor-role), I recommend teams to remove those permissions.

This can be done via an organization policy:

```terraform
resource "google_organization_policy" "remove_default_iam_grants" {
  org_id     = var.organization_id
  constraint = "constraints/iam.automaticIamGrantsForDefaultServiceAccounts"
  boolean_policy {
    enforced = false
  }
}
```

The consequence is that the process to create a GKE cluster can't fall back on the default Compute service account to perform necessary operations.  We therefore will create custom service accounts and assign the necessary permissions to these service accounts.  This will allow us to tightly control what these identities can do, making the overall architecture more secure.

## IAM

By using Shared VPC and creating a custom service account, this topic needs a bit more background information, as not everything is documented clearly on our public documentation.

Normally, GKE uses the default Compute service account, which receives Editor permissions on the project at creation time.  However, as mentioned in the introduction, this violates the principle of least privilege and we typically recommend not to use that.  So we will create a custom service account for the cluster.

In addition to this service account, enabling the Container API on the project will also result in two service accounts being created.   These are very important and shouldn’t be tampered with, as this will cause all operations to fail.

### Google Kubernetes Engine Service Account

When enabling the container-api, a service account is automatically created in the form of `service-[PROJECT_NMBR]@container-engine-robot.iam.gserviceaccount.com`.   This service account gives permissions to the Kubernetes Engine control plane to take action on the customer’s behalf on resources in the customer’s project.  Any action that touches customer project resources relies on this account (e.g. creating clusters, creating the peering connection with the network hosting the control plane).

From a Shared VPC perspective, the GKE control plane uses the robot account in the Host project to create firewall rules and routes in the network.  In the Service project(s), the service account is used for all other resources, i.e. instance templates and managed instance groups.  Additionally, the Service Robot from the Service project has to have the `roles/compute.networkUser`-role on the host project (either on a per subnet level or on the network itself), as otherwise it can’t deploy any virtual machines (i.e. nodes) in the Host network.

If you run `gcloud projects get-iam-policy [PROJECT_ID]`, you’ll notice that the role `roles/container.serviceAgent` is assigned to the service account.   You can run `gcloud iam roles describe roles/container.serviceAgent` to list all the permissions assigned to the role. 

### Firewall Rules
    
Firewall rules and routes are managed at the Host-project, so they can only be modified by the robot belonging to the Host project.  If teams are ok with this, they can grant the `Security Admin`-role to the Robot service account in the Service project.  
### Google APIs service account

Additionally to the previous service accounts, another one is created: `Google APIs service account`.  It takes the form of `PROJECT_NUMBER@cloudservices.gserviceaccount.com` and is used to call other Google APIs.  

It’s not entirely clear what this service account is being used for, but for example creating node pools fails if this service account doesn't have the necessary permissions.  Presumably because it's being used to create the managed instance groups for node pools. 

## Configuration

### Projects

To keep this configuration easy to manage, we will go for 3 projects, one host project and two service projects.  Later on we can extend this by adding more service projects, but for now this should be sufficient.

#### Host Project

```terraform
module "gke_host_project" {
  source  = "terraform-google-modules/project-factory/google"
  version = "~> 9.0"

  name                 = "${var.project_prefix}-host"
  random_project_id    = true
  org_id               = var.organization_id
  folder_id            = var.folder_id
  billing_account      = var.billing_account_id
  skip_gcloud_download = true

  activate_apis = [
    "container.googleapis.com"
  ]
}
```

### Service Projects

```terraform
module "gke_svc_one" {
  source  = "terraform-google-modules/project-factory/google"
  version = "~> 9.0"

  name                 = "${var.project_prefix}-svc-1"
  random_project_id    = true
  org_id               = var.organization_id
  billing_account      = var.billing_account_id
  folder_id            = var.folder_id
  skip_gcloud_download = true

  activate_apis = [
    "container.googleapis.com"
  ]
}

module "gke_svc_two" {
  source  = "terraform-google-modules/project-factory/google"
  version = "~> 9.0"

  name                 = "${var.project_prefix}-svc-2"
  random_project_id    = true
  org_id               = var.organization_id
  billing_account      = var.billing_account_id
  folder_id            = var.folder_id
  skip_gcloud_download = true

  activate_apis = [
    "container.googleapis.com"
  ]
}
```

The Container-API has to be enabled on both the host and service projects.  This will also cause the following APIs to be enabled:

```
bigquery.googleapis.com           
bigquerystorage.googleapis.com
compute.googleapis.com            
containerregistry.googleapis.com
iam.googleapis.com                
iamcredentials.googleapis.com 
monitoring.googleapis.com        
oslogin.googleapis.com            
pubsub.googleapis.com            
storage-api.googleapis.com
```
### Network

As show on the diagram, we will create one host network and two subnetworks.  Each subnetwork will be shared with one service project.  As our nodes won't have any public IP addresses, we enable Private Google Access on the subnet (`private_ip_google_access = true`).  To keep it simple, the subnets are located in two regions, `europe-west1` and `europe-west2`.  

//TODO: Add more information around IP address management.

```terraform
# Host Network
resource "google_compute_network" "gke_host_network" {
  project                 = module.gke_host_project.project_id
  name                    = "${var.network_name}-${random_id.randomizer.hex}"
  auto_create_subnetworks = false
  description             = "Host Network for GKE clusters"
}

# Subnetworks
resource "google_compute_subnetwork" "gke_host_subnet_1" {
  project                  = module.gke_host_project.project_id
  ip_cidr_range            = "10.0.4.0/22"
  name                     = "${var.network_name}-sn-euw1"
  network                  = google_compute_network.gke_host_network.self_link
  region                   = "europe-west1"
  private_ip_google_access = true

  secondary_ip_range {
    ip_cidr_range = "10.4.0.0/14"
    range_name    = "gke-pod-euw1-secondary"
  }

  secondary_ip_range {
    ip_cidr_range = "10.0.32.0/20"
    range_name    = "gke-svc-euw1-secondary"
  }
}

resource "google_compute_subnetwork" "gke_host_subnet_2" {
  project                  = module.gke_host_project.project_id
  name                     = "${var.network_name}-sn-euw2"
  network                  = google_compute_network.gke_host_network.self_link
  ip_cidr_range            = "172.16.4.0/22"
  region                   = "europe-west2"
  private_ip_google_access = true

  secondary_ip_range {
    ip_cidr_range = "172.20.0.0/14"
    range_name    = "gke-pod-euw2-secondary"
  }

  secondary_ip_range {
    ip_cidr_range = "172.16.16.0/20"
    range_name    = "gke-service-euw2-secondary"
  }
}

```



 