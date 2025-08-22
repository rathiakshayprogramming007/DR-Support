# Notebook Rehydration 🔄

### What is Rehydration?

When you create a Workbench (user-managed) or AI Notebook instance, its boot and data disks are **persistent disks (PDs)**. **Rehydration** is the process of retaining a PD after a VM is deleted or stopped and then re-attaching it to a new notebook instance. This allows you to preserve the data across different VM lifecycles.

---

### Terraform Implementation

To implement notebook rehydration with Terraform, you separate the lifecycle of the data disk from the Workbench instance.

#### 1. Create a Separate Data Disk

First, define the persistent disk as a separate resource. This disk will survive the deletion of the notebook VM.

```terraform
resource "google_compute_disk" "notebook_data" {
  name  = "notebook-data-disk"
  type  = "pd-ssd"
  size  = 200
  zone  = var.zone

  lifecycle {
    prevent_destroy = true
  }
}
```

The prevent_destroy = true argument in the lifecycle block is crucial. It ensures that Terraform will not delete this disk if you run terraform destroy or re-apply the configuration, making it the backbone of our rehydration strategy.

#### 2 Create the Workbench Instance with a Startup Script

Next, deploy the Workbench instance. we will use a startup script to automatically attach and mount the existing data disk every time the VM boots. Save the following script as attach-disk.sh in a scripts directory within our Terraform module

##### attach-disk.sh
```bash
#!/bin/bash
# attach-disk.sh
# Safe rehydration script for GCP Vertex AI Workbench (user-managed)

set -euo pipefail

DISK_NAME="notebook-data-disk"       # Change this if your disk name differs
MOUNT_DIR="/home/jupyter/notebook-data"
DEVICE_PATH="/dev/disk/by-id/google-${DISK_NAME}"
ZONE="${1}"  

echo "[INFO] Attaching disk ${DISK_NAME}..."

# Attach the disk if not already attached
if ! lsblk -o NAME,SERIAL | grep -q "${DISK_NAME}"; then
  echo "[INFO] Disk not attached, trying to attach..."
  gcloud compute instances attach-disk "$(hostname)" \
    --disk="${DISK_NAME}" \
    --zone="${ZONE}" || true
else
  echo "[INFO] Disk already attached."
fi

# Wait for device to appear
sleep 5

if [ ! -e "${DEVICE_PATH}" ]; then
  echo "[ERROR] Device ${DEVICE_PATH} not found!"
  exit 1
fi

# Check if the disk already has a filesystem
if ! blkid "${DEVICE_PATH}" >/dev/null 2>&1; then
  echo "[INFO] No filesystem found. Formatting disk..."
  mkfs.ext4 -F "${DEVICE_PATH}"
else
  echo "[INFO] Filesystem already exists. Skipping format."
fi

# Create mount point if missing
mkdir -p "${MOUNT_DIR}"

# Mount the disk if not already mounted
if ! mount | grep -q "${MOUNT_DIR}"; then
  echo "[INFO] Mounting ${DEVICE_PATH} to ${MOUNT_DIR}..."
  mount "${DEVICE_PATH}" "${MOUNT_DIR}"
  chown -R jupyter:jupyter "${MOUNT_DIR}"
else
  echo "[INFO] ${MOUNT_DIR} is already mounted."
fi

# Ensure persistence across reboots
if ! grep -q "${DEVICE_PATH}" /etc/fstab; then
  echo "[INFO] Adding ${DEVICE_PATH} to /etc/fstab..."
  echo "${DEVICE_PATH} ${MOUNT_DIR} ext4 defaults 0 2" >> /etc/fstab
fi

echo "[INFO] Disk ${DISK_NAME} successfully attached and mounted at ${MOUNT_DIR}."
```


In the main.tf file, configure the google_workbench_instance resource to run this script at boot.

```terraform
resource "google_workbench_instance" "notebook" {
  name     = "notebook-vm"
  location = var.zone

  gce_setup {
    .
    .
    .
    metadata = {
      startup-script = file("${path.module}/scripts/attach-disk.sh  ${var.zone}")
    }
  }
}
```

The startup script handles the rehydration process automatically:

1. It attaches the existing notebook-data-disk.
2. If the disk is brand new and has no filesystem, it formats it.
3. It mounts the disk at /home/jupyter/notebook-data.
4. It adds an entry to /etc/fstab to ensure the disk remains mounted after future reboots.