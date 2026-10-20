# ☸️ EKS Cluster & Persistent Storage Setup

*Technical Reference — Companion to the Day 2 Security remediation docs*

This guide covers provisioning the EKS cluster used by the CICD system, wiring up persistent storage (EBS CSI driver), and creating the namespaces, StorageClasses, PersistentVolumes (PVs), and PersistentVolumeClaims (PVCs) the workloads rely on — using only the **AWS CLI** and **kubectl**.

> ⚠️ All examples use placeholder values driven by environment variables (see below). Set the variables once and the commands run as-is — no manual find-and-replace.

## Overview

| **Layer**        | **Resource**                          | **Purpose**                                        |
|------------------|---------------------------------------|----------------------------------------------------|
| AWS              | EKS cluster + managed node group      | Control plane and worker nodes for CICD workloads  |
| AWS              | EBS CSI driver (EKS add-on) + IRSA    | Lets Kubernetes dynamically provision EBS volumes  |
| Kubernetes       | Namespaces                            | Logical isolation per environment / team           |
| Kubernetes       | StorageClass                          | Defines how volumes are dynamically provisioned    |
| Kubernetes       | PersistentVolume (PV)                 | Cluster storage resource (static or dynamic)       |
| Kubernetes       | PersistentVolumeClaim (PVC)           | A pod's request for storage                        |

## Prerequisites

- AWS CLI v2 and `kubectl` installed
- IAM permissions to create EKS clusters, node groups, IAM roles, OIDC providers, and VPC resources
- `curl` available (used to auto-detect your public IP)

---

## 0. Environment Variables

Set these once in your shell. Every command below reads from them, so you never edit a command by hand. Replace the values on the right with your own.

```bash
# --- Identity & region ---
export AWS_REGION=eu-west-1
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# --- Cluster ---
export CLUSTER_NAME=cicd-production
export K8S_VERSION=1.36

# --- Networking (created in section 1; two AZs) ---
export VPC_CIDR=10.0.0.0/16
export AZ_A=${AWS_REGION}a
export AZ_B=${AWS_REGION}b
export SUBNET_A_CIDR=10.0.1.0/24
export SUBNET_B_CIDR=10.0.2.0/24
# SUBNET_IDS and SECURITY_GROUP_IDS are captured into env vars in section 1.

# --- Your public IP, so the EKS API endpoint is reachable from this machine ---
# Auto-detect, or set it manually, e.g. export MY_PUBLIC_IP=203.0.113.25
export MY_PUBLIC_IP=46.171.92.161
export MY_PUBLIC_CIDR=${MY_PUBLIC_IP}/32

# --- IAM role names (created below) ---
export CLUSTER_ROLE_NAME=eksClusterRole
export NODE_ROLE_NAME=eksNodeGroupRole
export EBS_CSI_ROLE_NAME=AmazonEKS_EBS_CSI_DriverRole

# --- Node group (cheapest practical x86 worker nodes) ---
export NODEGROUP_NAME=cicd-workers
export NODE_INSTANCE_TYPE=t3.small              # burstable, low cost
export NODE_AMI_TYPE=AL2023_x86_64_STANDARD     # x86_64 AMI for t3
export NODE_DESIRED=3
export NODE_MIN=2
export NODE_MAX=5
export NODE_DISK_SIZE=50                         # root EBS volume per node (GiB)

# --- A static EBS volume used later; its AZ must match a node AZ ---
export STATIC_VOLUME_AZ=$AZ_A
```

> 💰 **Why `t3.small`?** T3 burstable instances are a low-cost general-purpose
> choice and run standard x86_64 images, so your existing CICD containers work
> without rebuilding for ARM. The cheaper `t4g.small` (Graviton/ARM64) is an
> option only if all your images are multi-arch/`arm64`.

> 🌐 **`MY_PUBLIC_IP`** is used to lock the EKS public API endpoint and the
> cluster security group down to your address only, so `kubectl` works from the
> command line without exposing the API to the whole internet. If your IP
> changes (new network, VPN toggled), re-run the `export` and update the
> endpoint/SG rule.

---

## 1. VPC, Subnets & Security Group

Build a minimal public VPC with two subnets (EKS requires at least two AZs) and
a security group scoped to your public IP so the cluster is reachable from the
CLI. Every ID is captured into an env var, so nothing is copied by hand.

### Create the VPC

```bash
export VPC_ID=$(aws ec2 create-vpc \
    --cidr-block "$VPC_CIDR" \
    --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=${CLUSTER_NAME}-vpc},{Key=Environment,Value=${CLUSTER_NAME}}]" \
    --query Vpc.VpcId --output text)

# EKS needs DNS hostnames + resolution enabled on the VPC
aws ec2 modify-vpc-attribute --vpc-id "$VPC_ID" --enable-dns-support '{"Value":true}'
aws ec2 modify-vpc-attribute --vpc-id "$VPC_ID" --enable-dns-hostnames '{"Value":true}'

echo "VPC: $VPC_ID"
```

### Create an internet gateway and attach it

```bash
export IGW_ID=$(aws ec2 create-internet-gateway \
    --tag-specifications "ResourceType=internet-gateway,Tags=[{Key=Name,Value=${CLUSTER_NAME}-igw}]" \
    --query InternetGateway.InternetGatewayId --output text)

aws ec2 attach-internet-gateway --vpc-id "$VPC_ID" --internet-gateway-id "$IGW_ID"
```

### Create two public subnets (one per AZ)

```bash
export SUBNET_A_ID=$(aws ec2 create-subnet \
    --vpc-id "$VPC_ID" \
    --cidr-block "$SUBNET_A_CIDR" \
    --availability-zone "$AZ_A" \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=${CLUSTER_NAME}-subnet-a},{Key=kubernetes.io/role/elb,Value=1}]" \
    --query Subnet.SubnetId --output text)

export SUBNET_B_ID=$(aws ec2 create-subnet \
    --vpc-id "$VPC_ID" \
    --cidr-block "$SUBNET_B_CIDR" \
    --availability-zone "$AZ_B" \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=${CLUSTER_NAME}-subnet-b},{Key=kubernetes.io/role/elb,Value=1}]" \
    --query Subnet.SubnetId --output text)

# Give nodes public IPs so they can reach the EKS API / pull images
aws ec2 modify-subnet-attribute --subnet-id "$SUBNET_A_ID" --map-public-ip-on-launch
aws ec2 modify-subnet-attribute --subnet-id "$SUBNET_B_ID" --map-public-ip-on-launch

# Combined list used by the cluster and node group commands later
export SUBNET_IDS=${SUBNET_A_ID},${SUBNET_B_ID}
echo "Subnets: $SUBNET_IDS"
```

### Route the subnets to the internet gateway

```bash
export ROUTE_TABLE_ID=$(aws ec2 create-route-table \
    --vpc-id "$VPC_ID" \
    --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=${CLUSTER_NAME}-rt}]" \
    --query RouteTable.RouteTableId --output text)

aws ec2 create-route \
    --route-table-id "$ROUTE_TABLE_ID" \
    --destination-cidr-block 0.0.0.0/0 \
    --gateway-id "$IGW_ID"

aws ec2 associate-route-table --route-table-id "$ROUTE_TABLE_ID" --subnet-id "$SUBNET_A_ID"
aws ec2 associate-route-table --route-table-id "$ROUTE_TABLE_ID" --subnet-id "$SUBNET_B_ID"
```

### Create the cluster security group (scoped to your public IP)

```bash
export SG_ID=$(aws ec2 create-security-group \
    --group-name "${CLUSTER_NAME}-cluster-sg" \
    --description "EKS cluster SG - CLI access from a single public IP" \
    --vpc-id "$VPC_ID" \
    --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=${CLUSTER_NAME}-cluster-sg}]" \
    --query GroupId --output text)

# Allow HTTPS (Kubernetes API / kubectl) from your public IP only
aws ec2 authorize-security-group-ingress \
    --group-id "$SG_ID" \
    --protocol tcp \
    --port 443 \
    --cidr "$MY_PUBLIC_CIDR"

# Allow the SG to talk to itself (control plane <-> nodes)
aws ec2 authorize-security-group-ingress \
    --group-id "$SG_ID" \
    --protocol -1 \
    --source-group "$SG_ID"

# Used by the cluster create command later
export SECURITY_GROUP_IDS=$SG_ID
echo "Security group: $SG_ID (ingress 443 from $MY_PUBLIC_CIDR)"
```

> 🔐 Only your `MY_PUBLIC_CIDR` can reach the API on port 443. To add a teammate
> or a changed IP, run another `authorize-security-group-ingress` with their
> `/32`. Avoid `0.0.0.0/0` — that exposes the API server to the whole internet.

---

## 2. IAM Roles for the Cluster and Nodes

EKS needs a cluster role; the managed node group needs a node role.

### Cluster role

```bash
cat > cluster-trust-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "eks.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role \
    --role-name "$CLUSTER_ROLE_NAME" \
    --assume-role-policy-document file://cluster-trust-policy.json

aws iam attach-role-policy \
    --role-name "$CLUSTER_ROLE_NAME" \
    --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy
```

### Node group role

```bash
cat > node-trust-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "ec2.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role \
    --role-name "$NODE_ROLE_NAME" \
    --assume-role-policy-document file://node-trust-policy.json

for POLICY in \
    AmazonEKSWorkerNodePolicy \
    AmazonEKS_CNI_Policy \
    AmazonEC2ContainerRegistryReadOnly ; do
  aws iam attach-role-policy \
      --role-name "$NODE_ROLE_NAME" \
      --policy-arn arn:aws:iam::aws:policy/$POLICY
done
```

---

## 3. Provision the EKS Cluster (AWS CLI)

The `--resources-vpc-config` enables the public API endpoint but restricts it to
your public IP, so `kubectl` works from the CLI without opening the API to the
internet.

```bash
aws eks create-cluster \
    --name "$CLUSTER_NAME" \
    --region "$AWS_REGION" \
    --kubernetes-version "$K8S_VERSION" \
    --role-arn arn:aws:iam::${AWS_ACCOUNT_ID}:role/${CLUSTER_ROLE_NAME} \
    --resources-vpc-config "subnetIds=${SUBNET_IDS},securityGroupIds=${SECURITY_GROUP_IDS},endpointPublicAccess=true,endpointPrivateAccess=true,publicAccessCidrs=${MY_PUBLIC_CIDR}" \
    --access-config authenticationMode=API \
    --tags Environment=${CLUSTER_NAME},ManagedBy=security-team
```

> 💡 `authenticationMode=API` enables EKS access entries — required by the
> [EKS Access Entries guide](../access-setup/eks-access-entries.md).
> `publicAccessCidrs=${MY_PUBLIC_CIDR}` ties the API endpoint to your IP; combined
> with the security group rule from section 1, the cluster is reachable only from
> your machine.

Wait for the cluster to become active, then update kubeconfig:

```bash
aws eks wait cluster-active --name "$CLUSTER_NAME" --region "$AWS_REGION"

aws eks update-kubeconfig --name "$CLUSTER_NAME" --region "$AWS_REGION"
kubectl get svc   # sanity check: the default kubernetes service responds

# rename context
kubectl config rename-context \
    arn:aws:eks:eu-west-1:257515924280:cluster/cicd-production \
    cicd-production

# use context
kubectl config use-context cicd-production
kubectl config current-context   # verify
```

### Create the managed node group (cheap t3.small instances)

```bash
# Split the comma-separated subnet list into space-separated args for the CLI
aws eks create-nodegroup \
    --cluster-name "$CLUSTER_NAME" \
    --region "$AWS_REGION" \
    --nodegroup-name "$NODEGROUP_NAME" \
    --node-role arn:aws:iam::${AWS_ACCOUNT_ID}:role/${NODE_ROLE_NAME} \
    --subnets ${SUBNET_IDS//,/ } \
    --instance-types "$NODE_INSTANCE_TYPE" \
    --ami-type "$NODE_AMI_TYPE" \
    --disk-size "$NODE_DISK_SIZE" \
    --scaling-config minSize=${NODE_MIN},maxSize=${NODE_MAX},desiredSize=${NODE_DESIRED} \
    --labels role=cicd-worker \
    --tags Environment=${CLUSTER_NAME}

aws eks wait nodegroup-active \
    --cluster-name "$CLUSTER_NAME" \
    --nodegroup-name "$NODEGROUP_NAME" \
    --region "$AWS_REGION"
```

### Verify the cluster

```bash
kubectl get nodes -o wide
kubectl get pods -n kube-system
```

---

## 4. Install the EBS CSI Driver (for dynamic EBS volumes)

Without the CSI driver, PVCs stay `Pending` because nothing can provision real storage.

### Step 1: Create / associate the OIDC provider (required for IRSA)

```bash
# Get the cluster's OIDC issuer URL
export OIDC_ISSUER=$(aws eks describe-cluster --name "$CLUSTER_NAME" \
    --region "$AWS_REGION" \
    --query "cluster.identity.oidc.issuer" --output text)

export OIDC_ID=${OIDC_ISSUER#https://}   # strip scheme for the provider ARN

# Create the IAM OIDC provider if it doesn't already exist
aws iam list-open-id-connect-providers \
    --query "OpenIDConnectProviderList[?contains(Arn, '${OIDC_ID##*/}')]" --output text \
  || aws iam create-open-id-connect-provider \
        --url "$OIDC_ISSUER" \
        --client-id-list sts.amazonaws.com \
        --thumbprint-list $(echo | openssl s_client -servername oidc.eks.${AWS_REGION}.amazonaws.com \
            -connect oidc.eks.${AWS_REGION}.amazonaws.com:443 2>/dev/null \
            | openssl x509 -fingerprint -noout 2>/dev/null \
            | sed 's/.*=//; s/://g')
```

### Step 2: Create the IRSA role for the EBS CSI controller

```bash
cat > ebs-csi-trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::${AWS_ACCOUNT_ID}:oidc-provider/${OIDC_ID}"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "${OIDC_ID}:aud": "sts.amazonaws.com",
        "${OIDC_ID}:sub": "system:serviceaccount:kube-system:ebs-csi-controller-sa"
      }
    }
  }]
}
EOF

aws iam create-role \
    --role-name "$EBS_CSI_ROLE_NAME" \
    --assume-role-policy-document file://ebs-csi-trust-policy.json

aws iam attach-role-policy \
    --role-name "$EBS_CSI_ROLE_NAME" \
    --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy
```

### Step 3: Install the add-on

```bash
aws eks create-addon \
    --cluster-name "$CLUSTER_NAME" \
    --region "$AWS_REGION" \
    --addon-name aws-ebs-csi-driver \
    --service-account-role-arn arn:aws:iam::${AWS_ACCOUNT_ID}:role/${EBS_CSI_ROLE_NAME}

aws eks wait addon-active \
    --cluster-name "$CLUSTER_NAME" \
    --region "$AWS_REGION" \
    --addon-name aws-ebs-csi-driver
```

### Verify the driver

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver
# Expected: ebs-csi-controller-* and ebs-csi-node-* pods Running
```

---

## 5. Namespaces

Separate CICD workloads from platform tooling. Save as `namespaces.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: cicd-apps
  labels:
    environment: cicd-production
    app.kubernetes.io/managed-by: security-team
---
apiVersion: v1
kind: Namespace
metadata:
  name: cicd-data
  labels:
    environment: cicd-production
    app.kubernetes.io/managed-by: security-team
---
apiVersion: v1
kind: Namespace
metadata:
  name: cicd-tooling
  labels:
    environment: cicd-production
    app.kubernetes.io/managed-by: security-team
```

Apply and verify:

```bash
kubectl apply -f namespaces.yaml
kubectl get namespaces -l environment=cicd-production
```

---

## 6. StorageClass

A StorageClass tells the EBS CSI driver how to provision volumes. Save as `storage-class.yaml`:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3
  labels:
    app.kubernetes.io/managed-by: security-team
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  fsType: ext4
  encrypted: "true"          # encrypt volumes at rest (recommended)
volumeBindingMode: WaitForFirstConsumer   # provision in the pod's AZ
reclaimPolicy: Delete        # use Retain for data you must not lose
allowVolumeExpansion: true
```

Apply and verify:

```bash
kubectl apply -f storage-class.yaml
kubectl get storageclass
# ebs-gp3 should show "(default)"
```

> 🔒 `encrypted: "true"` uses the account's default EBS KMS key unless you add a
> `kmsKeyId` parameter. `reclaimPolicy: Retain` keeps the EBS volume after the
> PVC is deleted — safer for stateful data, but you clean up volumes manually.

---

## 7. PersistentVolumeClaims (dynamic provisioning)

With a StorageClass in place, a PVC triggers automatic EBS volume creation — the normal pattern on EKS. Save as `pvc-dynamic.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: artifacts-cache
  namespace: cicd-apps
  labels:
    app.kubernetes.io/managed-by: security-team
spec:
  accessModes:
    - ReadWriteOnce          # EBS volumes are single-node (RWO)
  storageClassName: ebs-gp3
  resources:
    requests:
      storage: 20Gi
```

Apply and verify:

```bash
kubectl apply -f pvc-dynamic.yaml
kubectl get pvc -n cicd-apps
# STATUS is Pending until a pod mounts it (WaitForFirstConsumer)
```

### Consume the PVC in a pod

Save as `pod-with-pvc.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cache-writer
  namespace: cicd-apps
spec:
  containers:
    - name: app
      image: public.ecr.aws/docker/library/busybox:latest
      command: ["sh", "-c", "echo hello > /data/test.txt && sleep 3600"]
      volumeMounts:
        - name: cache
          mountPath: /data
  volumes:
    - name: cache
      persistentVolumeClaim:
        claimName: artifacts-cache
```

```bash
kubectl apply -f pod-with-pvc.yaml
kubectl get pvc -n cicd-apps          # now Bound
kubectl get pv                        # a PV was created automatically
kubectl exec -n cicd-apps cache-writer -- cat /data/test.txt   # hello
```

---

## Cleanup

```bash
# Delete workloads and claims (order matters for Retain volumes)
kubectl delete pod cache-writer -n cicd-apps
kubectl delete pvc artifacts-cache -n cicd-apps

# Tear down node group, then cluster, then IAM roles
aws eks delete-nodegroup --cluster-name "$CLUSTER_NAME" \
    --nodegroup-name "$NODEGROUP_NAME" --region "$AWS_REGION"
aws eks wait nodegroup-deleted --cluster-name "$CLUSTER_NAME" \
    --nodegroup-name "$NODEGROUP_NAME" --region "$AWS_REGION"

aws eks delete-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION"
aws eks wait cluster-deleted --name "$CLUSTER_NAME" --region "$AWS_REGION"
```

---

## Troubleshooting

### PVC stuck in `Pending`

- With `WaitForFirstConsumer`, this is normal until a pod mounts the PVC. Schedule a pod.
- Otherwise the EBS CSI driver is missing or its IRSA role lacks permissions:

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver
kubectl describe pvc <name> -n <namespace>   # read the Events section
```

### `failed to provision volume ... could not create volume in EC2`

- The `ebs-csi-controller-sa` service account is not bound to an IAM role with
  `AmazonEBSCSIDriverPolicy`. Re-check the IRSA setup in section 4.

### `kubectl` times out / connection refused

- Your public IP changed. Re-detect and update both the endpoint allow-list and
  the security group rule:

```bash
export MY_PUBLIC_IP=$(curl -s https://checkip.amazonaws.com)
export MY_PUBLIC_CIDR=${MY_PUBLIC_IP}/32

aws eks update-cluster-config --name "$CLUSTER_NAME" --region "$AWS_REGION" \
    --resources-vpc-config "endpointPublicAccess=true,publicAccessCidrs=${MY_PUBLIC_CIDR}"

aws ec2 authorize-security-group-ingress \
    --group-id "$SG_ID" --protocol tcp --port 443 --cidr "$MY_PUBLIC_CIDR"
```

### Node group fails: `exec format error` in pods

- A pod's image architecture doesn't match the node. `t3.small` is x86_64, so
  use x86_64 or multi-arch images. (If you switch to `t4g.*` Graviton nodes,
  set `NODE_AMI_TYPE=AL2023_ARM_64_STANDARD` and use `arm64`/multi-arch images.)

### Pod can't schedule: `volume node affinity conflict`

- A static PV's `nodeAffinity` AZ doesn't match any node, or no node exists in
  the volume's AZ. EBS volumes are zonal — the pod must run in the volume's AZ.

### `Multi-Attach error` on a PVC

- `ReadWriteOnce` EBS volumes attach to one node at a time. Don't schedule two
  pods on different nodes against the same PVC. For shared storage use EFS
  (`efs.csi.aws.com`) with `ReadWriteMany` instead.
```
