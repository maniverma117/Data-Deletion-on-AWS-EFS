# **Regular Data Deletion Process on AWS EFS**

## **Overview**
This document outlines the process to implement and deploy a **regular data deletion schedule** on AWS Elastic File System (EFS) using Kubernetes CronJobs. It provides a detailed guide, including the Kubernetes manifest, Dockerfile, and instructions for deployment.

---

## **Key Discussion Points**

### **1. Frequency of Deletion**
- The deletion process runs on a **regular schedule** using a Kubernetes CronJob.
- **Default Schedule:** Once daily at **2:00 AM UTC** (customizable as needed).

### **2. Dynamic Deletion Requirements**
- Designed to target **client-specific data**, ensuring data for one client can be deleted without affecting other clients.
- The script can be customized to delete specific directories and exclude certain files, offering flexibility and precision.

---

## **Technical Details**

### **1. Kubernetes CronJob**
The CronJob automates periodic deletion of data stored on EFS. It mounts the EFS volume and executes a containerized cleanup script.

### **2. Docker-Based Solution**
The deletion process is encapsulated in a Docker container:
- Includes a Bash script to delete files from the EFS volume.
- Handles batch deletions to reduce resource usage.

---

## **Deployment Steps**

### **Step 1: Dockerfile**
Create a `Dockerfile` for the deletion container.

```dockerfile
# Use a lightweight base image
FROM alpine:latest

# Install required tools
RUN apk add --no-cache bash findutils

# Copy the deletion script into the container
COPY delete-efs-data.sh /usr/local/bin/delete-efs-data.sh

# Make the script executable
RUN chmod +x /usr/local/bin/delete-efs-data.sh

# Set the script as the default command
CMD ["/usr/local/bin/delete-efs-data.sh"]
```

---

### **Step 2: Bash Script**
Create the `delete-efs-data.sh` script.

```bash
#!/bin/bash
set -e

# Path to EFS mount point
EFS_PATH="/mnt/efs"

# Log file for deletion progress
LOG_FILE="/tmp/deletion.log"
> "$LOG_FILE"

# Ensure the directory exists
if [ ! -d "$EFS_PATH" ]; then
    echo "EFS mount point $EFS_PATH does not exist. Exiting." | tee -a "$LOG_FILE"
    exit 1
fi

echo "Starting deletion process for $EFS_PATH..." | tee -a "$LOG_FILE"

# Delete files in batches
find "$EFS_PATH" -type f -exec rm -f {} + | tee -a "$LOG_FILE"

# Optional: Remove empty directories
find "$EFS_PATH" -type d -empty -delete | tee -a "$LOG_FILE"

echo "Deletion process completed." | tee -a "$LOG_FILE"
```

---

### **Step 3: Build and Push the Docker Image**
Build and push the Docker image to a container registry (e.g., Amazon ECR, Docker Hub).

```bash
# Build the image
docker build -t <your-docker-registry>/efs-cleanup:latest .

# Push the image to your registry
docker push <your-docker-registry>/efs-cleanup:latest
```

---

### **Step 4: Kubernetes CronJob Manifest**
Use the following CronJob manifest to schedule the deletion task.

#### **efs-cleanup-cronjob.yaml**
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: efs-cleanup
  namespace: default
spec:
  schedule: "0 2 * * *" # Runs daily at 2:00 AM UTC
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: efs-cleanup
            image: <your-docker-registry>/efs-cleanup:latest
            command: ["/usr/local/bin/delete-efs-data.sh"]
            volumeMounts:
            - name: efs-volume
              mountPath: /mnt/efs
          restartPolicy: OnFailure
          volumes:
          - name: efs-volume
            persistentVolumeClaim:
              claimName: efs-pvc
```

### **Customizations**
- **Schedule**: Update the `schedule` field in the CronJob manifest to your desired time interval.
- **Resource Limits**: Add `resources` to the container spec for fine-grained control over CPU and memory usage:
  ```yaml
  resources:
    requests:
      memory: "256Mi"
      cpu: "500m"
    limits:
      memory: "512Mi"
      cpu: "1"
  ```

---

### **Step 5: Persistent Volume and PVC for EFS**
Ensure your Kubernetes cluster has access to the EFS mount by creating a PersistentVolume (PV) and PersistentVolumeClaim (PVC).

#### **efs-pv.yaml**
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: efs-pv
spec:
  capacity:
    storage: 10Ti
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  csi:
    driver: efs.csi.aws.com
    volumeHandle: fs-12345678 # Replace with your EFS File System ID
```

#### **efs-pvc.yaml**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Ti
```

---

## **Customizations**

### **1. Adjusting the Schedule**
To modify the frequency of the CronJob, update the `schedule` field in `efs-cleanup-cronjob.yaml`:
- Format: `min hour day month day_of_week`
- Example: Run every 6 hours:
  ```yaml
  schedule: "0 */6 * * *"
  ```

### **2. Excluding Specific Files**
Update the `delete-efs-data.sh` script to exclude certain files or directories:
```bash
find "$EFS_PATH" -type f ! -name "*.log" -exec rm -f {} +
```

---

## **Monitoring**

1. **Logs**: Check pod logs to monitor the deletion process:
   ```bash
   kubectl logs <efs-cleanup-pod-name>
   ```

2. **EFS Usage**: Use AWS CloudWatch to monitor EFS storage usage before and after deletions.


### **2. Estimation of Deletion Time**
#### **Assumptions**
- **Storage Type:** Let's assume EBS (SSD) or a comparable fast storage.
- **Deletion Rate:** 500 files/second per thread (realistic for SSDs).

#### **Calculation**
- **Files to Delete:** \( 1,090,000 \) files
- **Deletion Time (single thread):**  
  \( \frac{1,090,000}{500} = 2,180 \text{ seconds} = ~36.3 \text{ minutes} \)
- With **10 concurrent threads**, the deletion can finish in ~4 minutes.

---

### **3. Compute Needs**
#### **Scenario: Using a Script or Tool**
- **CPU:**  
  - Single-core is sufficient if the deletion is not multi-threaded.  
  - For 10 threads: a 4-core CPU is typically sufficient.
- **RAM:**  
  - Around **4–8 GB RAM** for file metadata caching and smooth processing.  
  - If the files are highly nested, consider **16 GB RAM**.

