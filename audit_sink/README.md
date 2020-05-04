```terraform
module "audit_log_project" {
  source = "git@github.com:debakkerb/tf-modules//10_-_standalone/project"

  billing_account_id = local.billing_account_id
  folder_id          = local.audit_folder_id
  project_name = "org-audit-sink"

  project_services = [
    "bigquery.googleapis.com",
    "storage-component.googleapis.com"
  ]

}

resource "google_storage_bucket" "bdb_org_audit_sink_bucket" {
  project = module.audit_log_project.project_id

  name               = "bdb-org-audit-sink"
  bucket_policy_only = true

  location = "EU"

  force_destroy = true
}

resource "google_logging_organization_sink" "gcs_audit_org_sink" {
  org_id      = local.organization_id
  name        = "audit-organisation-logs-gcs"
  destination = "storage.googleapis.com/${google_storage_bucket.bdb_org_audit_sink_bucket.id}"

  include_children = true

  filter = "logName=(organizations/${local.organization_id}/logs/cloudaudit.googleapis.com%2Factivity OR organizations/${local.organization_id}/logs/cloudaudit.googleapis.com%2Fdata_access OR organizations/${local.organization_id}/logs/cloudaudit.googleapis.com%2Fsystem_event)"
}

resource "google_project_iam_member" "gcs_log_writer" {
  project = module.audit_log_project.project_id

  member = google_logging_organization_sink.gcs_audit_org_sink.writer_identity
  role   = "roles/storage.objectCreator"
}

resource "google_logging_organization_sink" "bq_audit_org_sink" {
  org_id      = local.organization_id
  name        = "audit-organisation-logs-bq"
  destination = "bigquery.googleapis.com/projects/${module.audit_log_project.project_id}/datasets/${google_bigquery_dataset.bdb_org_audit_sink_dataset.dataset_id}"

  include_children = true

  filter = "logName=(organizations/${local.organization_id}/logs/cloudaudit.googleapis.com%2Factivity OR organizations/${local.organization_id}/logs/cloudaudit.googleapis.com%2Fdata_access OR organizations/${local.organization_id}/logs/cloudaudit.googleapis.com%2Fsystem_event)"
}

resource "google_project_iam_member" "bq_log_writer" {
  project = module.audit_log_project.project_id

  member = google_logging_organization_sink.bq_audit_org_sink.writer_identity
  role   = "roles/bigquery.dataEditor"
}

resource "google_bigquery_dataset" "bdb_org_audit_sink_dataset" {
  project = module.audit_log_project.project_id

  dataset_id  = "bdb_audit_sink"
  description = "BigQuery Dataset that contains all the logs exported at Organization level."
  location    = "EU"

  access {
    role          = "OWNER"
    user_by_email = "bjorn@bdb-tst.co.uk"
  }

}
```