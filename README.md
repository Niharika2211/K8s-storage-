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
- The **reclaim policy** defines what happens to the PV when the PVC is deleted (Retain, Recycle, Delete).

### **Persistent Volume Claim (PVC)**
- A **PVC is a request for storage** by a user or application.
- It binds to a PV and allows pods to use the storage.
- Users do not need to know the details of the underlying storage.

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

## **🔹 Step 6: Deploy PVC and Pod**
Apply the YAML files:
```sh
kubectl apply -f storageclass.yaml
kubectl apply -f pvc.yaml
kubectl apply -f pod.yaml
```
Verify storage provisioning:
```sh
kubectl get pvc
kubectl get pv
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

## **🔥 FAQs for Interviews**

### **1. Do I need to create a PV manually in dynamic provisioning?**
❌ **No**. In **dynamic provisioning**, Kubernetes automatically creates a PV when the PVC is created.

### **2. Where should I install the EBS CSI driver?**
✅ It should be installed in the **`kube-system` namespace**.

### **3. What happens if the IAM policy is not attached?**
❌ The **EBS CSI driver will fail to provision storage**. You will see errors related to volume creation failure.

### **4. Where do we define the Reclaim Policy?**
✅ **Reclaim Policy is defined in the PV**, not in the PVC.
```sh
kubectl get pv -o wide  # Check reclaim policy of PVs
```

### **5. How do I check which StorageClasses are available?**
```sh
kubectl get storageclass
```

---

📌 **Author:** Niharika (*DevOps Engineer*)

