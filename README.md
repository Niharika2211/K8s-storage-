# **EKS Persistent Storage with AWS EBS CSI Driver**

This guide explains how to set up **EBS-backed persistent storage in an EKS cluster** using the **AWS EBS CSI Driver**. This setup supports both **static** and **dynamic provisioning**.

## **🛠 Prerequisites**
- An **AWS Account**
- **eksctl** installed
- **kubectl** installed
- **Helm** installed

---

## **📌 What is Kubernetes Storage?**
Kubernetes provides a way to manage persistent storage using **Persistent Volumes (PV)** and **Persistent Volume Claims (PVC)**.

### **Persistent Volume (PV)**
- A **PV is a storage resource** in the cluster that an administrator provisions.
- It can be backed by cloud storage (EBS, NFS, etc.) or on-prem solutions.
- The **reclaim policy** defines what happens to the PV when the PVC is deleted (**Retain, Recycle, Delete**).

### **Persistent Volume Claim (PVC)**
- A **PVC is a request for storage** by a user or application.
- It binds to a PV and allows pods to use the storage.
- Users do not need to know the details of the underlying storage.

### **Reclaim Policy**
- Defined in the **Persistent Volume (PV)**.
- Determines the fate of the PV after PVC deletion:
  - **Retain**: Keeps the PV even if the PVC is deleted.
  - **Delete**: Automatically deletes the PV along with the PVC.
  - **Recycle**: Cleans the PV and makes it available for reuse (deprecated in Kubernetes 1.20+).

### **Access Modes**
Defines how a PV can be accessed:
- **ReadWriteOnce (RWO)**: Can be mounted as read/write by a single node.
- **ReadOnlyMany (ROX)**: Can be mounted as read-only by multiple nodes.
- **ReadWriteMany (RWX)**: Can be mounted as read/write by multiple nodes.

---

## **🔹 Dynamic vs. Static Provisioning**

### **Static Provisioning**
- **Pre-created PVs** by an administrator.
- PVCs bind to these PVs when a request matches.
- Useful when exact control over storage is needed.
- **Example:** Manually creating an EBS volume and referencing it in a PV.

### **Dynamic Provisioning**
- Kubernetes **automatically creates PVs** when a PVC is requested.
- Requires a **StorageClass** to define storage properties.
- **No need to manually create PVs**.
- **Recommended for cloud-native environments**.
- **Example:** When a pod requests storage, Kubernetes provisions an EBS volume dynamically.

---

## **🚀 Step 1: Create an EKS Cluster**
Create an EKS cluster with worker nodes that support EBS:
```sh
eksctl create cluster --name my-eks-cluster --region us-east-1 --nodegroup-name my-nodes --nodes 2
```
👉 **Ensure IAM roles and policies are configured correctly.**

---

## **🔹 Step 2: Install the AWS EBS CSI Driver**
Deploy the EBS CSI driver in the `kube-system` namespace:
```sh
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm repo update
helm install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
  --namespace kube-system \
  --set enableVolumeScheduling=true \
  --set enableVolumeResizing=true \
  --set enableVolumeSnapshot=true
```
👉 **Required for both static & dynamic provisioning**.

---

## **🔹 Step 3: Attach IAM Role for EBS CSI Driver**
Attach the **AmazonEBSCSIDriverPolicy** to your **node group IAM role**:
```sh
aws iam attach-role-policy \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --role-name <your-nodegroup-role>
```
👉 **This is mandatory for EBS storage provisioning.**

---

## **📌 Step 4: Static Provisioning (Optional)**
If you want to **manually create PV & PVC**, use the following YAML files.

### **Persistent Volume (PV)**
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  csi:
    driver: ebs.csi.aws.com
    volumeHandle: vol-xxxxxxxx  # Replace with your EBS Volume ID
```

### **Persistent Volume Claim (PVC)**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: manual
```

---

## **📌 Step 5: Dynamic Provisioning (Recommended)**
For dynamic provisioning, create **StorageClass, PVC, and Pod**.

### **StorageClass**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
```

### **Persistent Volume Claim (PVC)**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: ebs-sc
```

### **Pod Using PVC**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - mountPath: "/data"
          name: storage-volume
  volumes:
    - name: storage-volume
      persistentVolumeClaim:
        claimName: ebs-claim
```

---

## **✅ Interview-Ready Summary**
| Step | Action | Required for Static? | Required for Dynamic? |
|------|--------|----------------------|----------------------|
| **1** | Create EKS cluster | ✅ Yes | ✅ Yes |
| **2** | Install EBS CSI driver | ✅ Yes | ✅ Yes |
| **3** | Attach IAM Role | ✅ Yes | ✅ Yes |
| **4** | Create PV manually | ✅ Yes | ❌ No |
| **5** | Create StorageClass | ❌ No | ✅ Yes |
| **6** | Create PVC | ✅ Yes | ✅ Yes |
| **7** | Create Pod using PVC | ✅ Yes | ✅ Yes |
| **8** | Verify storage | ✅ Yes | ✅ Yes |

---

📌 **Author:** Niharika (*DevOps Engineer*)

