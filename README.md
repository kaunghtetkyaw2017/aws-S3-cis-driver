# Managing Persistent Storage with Amazon S3 on Amazon EKS

### **1. Create S3 bucket with unique name.**

## **2. Save the Json file with `s3-access-policy.json`:**
   > Replace `amzn-s3-demo-bucket1` with your own Amazon S3 bucket name.

```json
{
   "Version": "2012-10-17",
   "Statement": [
        {
            "Sid": "MountpointFullBucketAccess",
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::amzn-s3-demo-bucket1"
            ]
        },
        {
            "Sid": "MountpointFullObjectAccess",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:AbortMultipartUpload",
                "s3:DeleteObject"
            ],
            "Resource": [
                "arn:aws:s3:::amzn-s3-demo-bucket1/*"
            ]
        }
   ]
}
```

## **3. Create policy with `s3-access-policy.json`:**
```bash
aws iam create-policy \
    --policy-name AmazonS3CSIDriverPolicy \
    --policy-document file://s3-access-policy.json
```

## **4. Save the Json file with `aws-s3-csi-driver-trust-policy.json`:**
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::<Account IC>:oidc-provider/oidc.eks.ap-southeast-1.amazonaws.com/id/<oidc id>"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "oidc.eks.ap-southeast-1.amazonaws.com/id/<oidc id>:sub": "system:serviceaccount:kube-system:s3-csi-driver-sa",
                    "oidc.eks.ap-southeast-1.amazonaws.com/id/<oidc id>:aud": "sts.amazonaws.com"
                }
            }
        }
    ]
}
```

## **5. Create the role using AWS CLI:**
```bash
aws iam create-role \
    --role-name AmazonEKS_S3_CSI_DriverRole \
    --assume-role-policy-document file://"aws-s3-csi-driver-trust-policy.json"
```

## **6. Attach `AmazonS3CSIDriverPolicy` ARN to the role:**
```bash
aws iam attach-role-policy \
    --role-name AmazonEKS_S3_CSI_DriverRole \
    --policy-arn arn:aws:iam::881490128348:policy/AmazonS3CSIDriverPolicy
```

## **7. Associate role with service account:**
```bash
eksctl create iamserviceaccount \
  --name s3-csi-controller-sa \
  --namespace kube-system \
  --cluster your-cluster-name \
  --attach-role-arn arn:aws:iam::aws-account-id:role/AmazonEKS_S3_CSI_DriverRole \
  --approve \
  --override-existing-serviceaccounts
```

## **8. Add Amazon S3 CSI add-on:**
```bash
eksctl create addon \
     --name aws-mountpoint-s3-csi-driver \
     --cluster my-cluster \
     --service-account-role-arn arn:aws:iam::111122223333:role/AmazonEKS_S3_CSI_DriverRole 
     --force

# Or using aws cli
aws eks create-addon \
    --cluster-name my-cluster \
    --addon-name aws-mountpoint-s3-csi-driver \
    --service-account-role-arn arn:aws:iam::111122223333:role/AmazonEKS_S3_CSI_DriverRole
```

## **9. Configure the Storage Class:**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: s3-csi
provisioner: s3.csi.aws.com
parameters:
  bucketName: uat-bk
  region: ap-southeast-1
  type: standard
reclaimPolicy: Retain
volumeBindingMode: Immediate
```

## **10. Create a PersistentVolume:**
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: s3-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  csi:
    driver: s3.csi.aws.com
    volumeHandle: uat-bk
    volumeAttributes:
      bucketName: uat-bk
      region: ap-southeast-1
```

## **11. Create a PersistentVolumeClaim:**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: s3-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  volumeName: s3-pv
```

## **12. Deploy a Sample Application:**

Write data in pod1:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: s3-test-pod-1
spec:
  containers:
  - name: nginx
    image: nginx
    command: ["/bin/sh"]
    args: ["-c", "echo 'Testing S3 CSI driver' > /data/test.txt && tail -f /dev/null"]
    volumeMounts:
    - name: s3-volume
      mountPath: /data
  volumes:
  - name: s3-volume
    persistentVolumeClaim:
      claimName: s3-pvc
```

Check data from pod2:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: s3-test-pod-2
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: s3-volume
      mountPath: /data
  volumes:
  - name: s3-volume
    persistentVolumeClaim:
      claimName: s3-pvc
```

```bash
kubectl exec -it s3-test-pod-2 -- ls /data
```
