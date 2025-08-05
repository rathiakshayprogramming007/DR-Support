Google Doc Link:

https://docs.google.com/document/d/1EoK2PVAhklR-VDJZnXa--VuVxjjnReorBGmjHxuikkE/edit?usp=sharing

-----> "Pipeline" <-----

1. Design Components for DR & HA
  a. Pipeline Definition Storage (Portable): Store pipeline YAML or JSON specs in GCS (dual-region or multi-region) and Artifact Registry.
  b. Use object versioning and lifecycle rules for retention.

2. Replication
  a. Store the python/component code in the github/VCS(version control system)
  b. Provision Pipeline using Terraform for multiregion deployment

3. Failover
  a. In-case of failure, switch to DR region for pipeline deployment using terraform

-----> Vertex AI Predication <-----

1. Cross-region Model Replication
Vertex AI models are bound to a single region.
To prepare for disasters, regularly export your trained models to a GCS bucket configured with multi-region or dual-region replication. Then, import the model into your DR region using gcloud, Terraform, or Python SDK. This ensures the model is available in case the primary region goes down.

2. Endpoint & Deployment Recovery
Endpoints that serve models for prediction are also regional. 
define endpoints in the DR region ahead of time, and deploy the replicated model there. Use Terraform to script this so you can quickly spin up endpoints without manual effort during recovery.

3. Backup Frequency & Synchronization
Automate regular backups of model versions using Cloud Scheduler + Cloud Functions or Cloud Build. Trigger a model export after each training job or at scheduled intervals (e.g., daily). These backups are saved in GCS and kept in sync with the DR region to avoid stale deployments during failover.

4. Infrastructure as Code (Terraform)
Manage all your model and endpoint infrastructure using Terraform modules, parameterizing the region to support both primary and DR deployments. Store Terraform state securely (e.g., Terraform Cloud or GCS backend) and ensure changes in primary are reproducible in DR.

5. Failover & Traffic Switching
Failover requires your application to be aware of both regions. Implement a traffic-switching mechanism, like a config-driven endpoint selection (from Firestore or Secret Manager), or use a proxy/load balancer to route requests. You must plan how to update clients to use the DR endpoint with minimal downtime.


-----> Artifact Registry ----->

1. Cross-Region Replication
  Artifact Registry is regional, meaning images/packages are stored in a specific region.
  You must replicate your artifacts to another region manually or via CI/CD pipelines.
  Use gcloud artifacts repositories list and gcloud artifacts docker images copy or equivalent to mirror images.

 Summary:
  Replicate container images or packages to a DR region.
  Use Cloud Build or CI/CD pipelines to automate mirroring across regions.
  Tag images by region/environment (e.g., gcr.io/region-artifact/image:us).



 2. Backup Strategy
  Enable image version retention policies.
  Store copies of critical images in Cloud Storage as tarballs (use docker save).
  Periodically verify image availability in both regions.

  Summary:
  Retain multiple versions per image.
  Export critical image versions to Cloud Storage for backup.
  Validate that backups can be restored into Artifact Registry.



3. Terraform & CI/CD Automation
  Use Terraform to define Artifact Registry repositories in both regions.
  CI/CD should push images to both us-central1 and asia-south1, for example.
  Use google_artifact_registry_repository resource and mirror settings.

  Summary:
  Define and manage repositories as code.
  Push images/packages to both primary and DR region.
  Automate cross-region publishing via pipelines.



4. Failover Plan
  Maintain alternate image URLs in deployment configs.
  During failover, update workload deployments to point to DR region repo.
  Optionally use Artifact Registry Virtual Repositories to unify access.
  
  Summary:
  Store DR image URLs in a config store (e.g., Secret Manager or Firestore).
  Automate image switch during failover.
  Optionally unify regions using virtual repos or proxies.


-----> Vertex AI Model Registry <-----

1. Model Artifact Backup (GCS + Metadata)
  Export model artifacts to a multi-region or dual-region GCS bucket.
  Automate exports after training completion.

 Summary:
  Save models in GCS with regional redundancy.
  Backup metadata in structured format.
  Automate this as part of your training pipeline.

 2. Replicate Model in DR Region
  Use gcloud ai models upload or Terraform to import the model in the DR region from GCS.
  Replicate model name, labels, and versions to mirror the original.

  Summary:
  Import the model into the DR region regularly.
  Ensure model versioning stays in sync across regions.
  Use Terraform or Python SDK to automate.

3. Terraform & DR Infrastructure
  Maintain Terraform configuration for both regions.
  Use modules or for_each blocks to provision models and endpoints in DR.
  Keep the DR infra in code so it's always ready to deploy.

 Summary:
  Model DR region definitions in Terraform.
  Sync state using Terraform Cloud or GCS backend.
  Update model versions using CI/CD pipelines.


 4. DR Testing for Model Registry
  Simulate model deletion or primary region outage.
  Test that DR model can be deployed and served via its endpoint.
  Validate model behavior (prediction accuracy, latency).

 Summary:
  Conduct DR drills for model availability.
  Confirm model registry can serve requests in the DR region.
  Validate data + endpoint readiness during simulated failure.

