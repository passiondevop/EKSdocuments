# EKS Hands-On Guide (AWS Console + Terminal)

Learn Amazon EKS manually from the AWS Console. You already know Kubernetes, so this guide focuses on the **AWS side**: cluster, IAM, nodes, add-ons, IRSA, Pod Identity, autoscaling.

> Region used in examples: `ap-south-1` (Mumbai). Cluster name: `my-eks`. Change as you like.
> **Cost warning:** EKS control plane is about $0.10/hour, plus EC2 nodes, plus NAT Gateway. Do the **Cleanup** section (Part 14) when you finish each session.

---

## Part 0: Order of work (big picture)

1. Setup terminal tools + AWS credentials
2. IAM role for the cluster
3. IAM role for worker nodes
4. VPC (networking)
5. Create the EKS cluster
6. Give yourself access + connect kubectl
7. Create a node group
8. Add-ons (VPC CNI, CoreDNS, kube-proxy, Pod Identity Agent, GuardDuty, etc.)
9. IRSA
10. Pod Identity
11. Cordon / drain / uncordon
12. Autoscaling (ASG + Cluster Autoscaler)
13. Troubleshooting
14. Cleanup

---

## Part 1: Terminal setup and AWS credentials

### 1.1 Install tools

- **AWS CLI v2**: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html
- **kubectl**: https://kubernetes.io/docs/tasks/tools/
- **helm** (needed later): https://helm.sh/docs/intro/install/

Check:

```bash
aws --version
kubectl version --client
helm version
```

### 1.2 Create access keys (for learning)

1. AWS Console → **IAM** → **Users** → Create user (e.g. `eks-admin`).
2. Attach policy `AdministratorAccess` (only for a personal learning account).
3. Open the user → **Security credentials** → **Create access key** → use case **CLI**.
4. Copy the **Access key ID** and **Secret access key** (shown only once).

### 1.3 Save credentials so they persist

```bash
aws configure --profile eks-lab
```

It asks:

```
AWS Access Key ID: AKIA...
AWS Secret Access Key: ****
Default region name: ap-south-1
Default output format: json
```

This **saves** to files on disk:

- `~/.aws/credentials` (keys)
- `~/.aws/config` (region, output)

Use the profile:

```bash
export AWS_PROFILE=eks-lab
aws sts get-caller-identity
```

If `get-caller-identity` prints your Account and ARN, the terminal is connected to AWS.

### 1.4 Why credentials "won't save"

| Problem | Reason | Fix |
|---|---|---|
| Works, but gone after closing terminal | You used `export AWS_ACCESS_KEY_ID=...` (lives only in that session) | Use `aws configure --profile eks-lab` (writes files) |
| Profile not picked up in new terminal | `AWS_PROFILE` is also session-only | Add `export AWS_PROFILE=eks-lab` to `~/.bashrc` or `~/.zshrc`, then `source ~/.bashrc` |
| `Unable to locate credentials` | No profile selected or wrong name | `aws configure list-profiles`, then `export AWS_PROFILE=<name>` |
| `ExpiredToken` | Temporary or SSO credentials expired | Re-login (see below) |
| Different user than expected | Old env vars override the profile | `unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN` |

SSO login (if your account uses IAM Identity Center):

```bash
aws configure sso
aws sso login --profile eks-lab
```

### 1.4a Alternative: AWS CloudShell

Top bar of the console → CloudShell icon. `aws` and `kubectl` basics are preinstalled and credentials are already set from your console login. Your `$HOME` persists across sessions, so it is good for practice.

### 1.5 Connect terminal to the cluster (after the cluster exists)

```bash
aws eks update-kubeconfig --region ap-south-1 --name my-eks --profile eks-lab
```

This writes the cluster entry into `~/.kube/config` and **saves** it. Check:

```bash
kubectl config get-contexts
kubectl get svc
kubectl get nodes
```

### 1.6 How to exit (all the "exit" cases)

| Situation | Command |
|---|---|
| Leave terminal / shell | `exit` or `Ctrl + D` |
| Leave a pod shell (`kubectl exec -it ... -- sh`) | `exit` or `Ctrl + D` |
| Exit `vim` and save | `Esc`, then `:wq` Enter |
| Exit `vim` without saving | `Esc`, then `:q!` Enter |
| Exit `nano` | `Ctrl + X`, then `Y`, Enter |
| Stop a running command (like `kubectl logs -f`, `get -w`) | `Ctrl + C` |
| Stop using temporary env credentials | `unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN AWS_PROFILE` |
| Forget the cluster from kubectl | `kubectl config delete-context <name>` and `kubectl config delete-cluster <name>` |
| Remove saved AWS keys completely | Delete the profile block from `~/.aws/credentials` and `~/.aws/config` |

---

## Part 2: IAM role for the EKS cluster

The control plane needs a role so EKS can manage AWS resources for you.

1. Console → **IAM** → **Roles** → **Create role**.
2. Trusted entity type: **AWS service**.
3. Use case: search **EKS** → choose **EKS - Cluster** → Next.
4. Policy: `AmazonEKSClusterPolicy` (already selected).
5. Role name: `eksClusterRole` → **Create role**.

---

## Part 3: IAM role for worker nodes

1. IAM → Roles → Create role.
2. Trusted entity: **AWS service** → use case **EC2** → Next.
3. Attach these three policies:
   - `AmazonEKSWorkerNodePolicy`
   - `AmazonEC2ContainerRegistryReadOnly` (or `AmazonEC2ContainerRegistryPullOnly`)
   - `AmazonEKS_CNI_Policy`
4. Role name: `eksNodeRole` → Create role.

> For learning, attaching `AmazonEKS_CNI_Policy` to the node role is fine. In production, give it to the `aws-node` service account through IRSA or Pod Identity instead (Parts 9 and 10).

---

## Part 4: VPC (networking)

EKS needs subnets in **at least 2 Availability Zones**.

Easiest path in the console:

1. Console → **VPC** → **Create VPC** → choose **VPC and more**.
2. Name tag: `eks-vpc`. IPv4 CIDR: `10.0.0.0/16`.
3. Number of AZs: **2**. Public subnets: **2**. Private subnets: **2**.
4. NAT gateways: **In 1 AZ** (cheaper for learning).
5. VPC endpoints: **None** (or S3 Gateway, free).
6. Enable **DNS hostnames** and **DNS resolution**.
7. Create VPC.

Subnet tags that help Kubernetes load balancers (add in VPC → Subnets → Tags):

| Subnet type | Tag key | Value |
|---|---|---|
| Public | `kubernetes.io/role/elb` | `1` |
| Private | `kubernetes.io/role/internal-elb` | `1` |

---

## Part 5: Create the EKS cluster

1. Console → **Elastic Kubernetes Service** → **Clusters** → **Create cluster**.
2. Choose **Custom configuration**. Turn **OFF** "Use EKS Auto Mode" (you want to learn the manual way).
3. **Step 1: Configure cluster**
   - Name: `my-eks`
   - Cluster IAM role: `eksClusterRole`
   - Kubernetes version: pick a standard-support version
   - Upgrade policy: Standard
   - Cluster access: **Allow cluster administrator access** (gives the creator admin)
   - Authentication mode: **EKS API and ConfigMap**
   - Next
4. **Step 2: Networking**
   - VPC: `eks-vpc`
   - Subnets: select the **private** subnets (public ones optional)
   - Security groups: leave default (EKS creates the cluster SG)
   - IP family: IPv4
   - Cluster endpoint access: **Public and private** (so your laptop can reach it)
   - Next
5. **Step 3: Observability** → keep defaults (optionally enable Control plane logging: API, audit, authenticator) → Next.
6. **Step 4: Add-ons** → keep `kube-proxy`, `CoreDNS`, `Amazon VPC CNI`. Also tick **EKS Pod Identity Agent**. Next.
7. **Step 5: Add-on settings** → keep default versions → Next.
8. Review → **Create**.

Wait about 10 minutes until Status = **Active**.

> CoreDNS stays *Pending* until nodes exist. That is normal.

---

## Part 6: Access and connect kubectl

### 6.1 Check who has access

Cluster → **Access** tab → **IAM access entries**. The identity that created the cluster is already an admin.

### 6.2 Give another IAM user or role access

1. Cluster → **Access** → **Create access entry**.
2. IAM principal ARN: pick the user or role.
3. Type: Standard.
4. Next → Access policy: `AmazonEKSClusterAdminPolicy` (full) or `AmazonEKSViewPolicy` (read only).
5. Access scope: Cluster (or Kubernetes namespace).
6. Create.

### 6.3 Connect from terminal

```bash
export AWS_PROFILE=eks-lab
aws eks update-kubeconfig --region ap-south-1 --name my-eks
kubectl get svc
```

You should see the `kubernetes` service. If you get `Unauthorized`, your IAM identity has no access entry (see 6.2) or you are using a different profile than the one that created the cluster.

---

## Part 7: Create a managed node group

1. EKS → Clusters → `my-eks` → **Compute** tab → **Node groups** → **Add node group**.
2. **Configure node group**
   - Name: `ng-general`
   - Node IAM role: `eksNodeRole`
   - Next
3. **Compute and scaling configuration**
   - AMI type: **Amazon Linux 2023 (x86_64) Standard**
   - Capacity type: **On-Demand** (Spot is cheaper, may be interrupted)
   - Instance type: `t3.small` (practice) 
   - Disk size: `50` GiB
   - Scaling config: **Desired 2, Minimum 1, Maximum 5**
   - Update config: Max unavailable = **1**
   - Next
4. **Networking**
   - Subnets: the **private** subnets
   - Remote access: off (use SSM instead if you need a shell)
   - Next
5. Review → **Create**.

Wait 3 to 5 minutes, then:

```bash
kubectl get nodes -o wide
```

Both nodes should be `Ready`.

### Other compute options (just know them)

| Type | What it is |
|---|---|
| Managed node group | AWS creates and manages EC2 + an Auto Scaling Group for you |
| Self-managed nodes | You create the launch template + ASG and join nodes yourself |
| Fargate profile | Serverless pods; Compute tab → Fargate profiles → Add |
| EKS Auto Mode | AWS manages nodes, networking, storage addons for you |
| Karpenter | Open source node provisioner (see Part 12) |

---

## Part 8: Add-ons in the console

### 8.1 Where to add

Cluster → **Add-ons** tab → **Get more add-ons** → tick the add-on → Next → choose version and (if needed) IAM role → Next → **Create**.

### 8.2 Core add-ons

| Add-on | Purpose |
|---|---|
| `kube-proxy` | Service networking on every node |
| `CoreDNS` | Cluster DNS |
| `Amazon VPC CNI` | Gives pods VPC IP addresses |
| `EKS Pod Identity Agent` | Needed for Pod Identity (Part 10) |
| `Amazon EBS CSI Driver` | Persistent volumes on EBS |
| `Metrics Server` | `kubectl top`, HPA |
| `Amazon CloudWatch Observability` | Container Insights logs and metrics |

### 8.3 Amazon VPC CNI add-on

Already installed with the cluster. To tune it:

1. Add-ons → click **Amazon VPC CNI** → **Edit**.
2. Pick a version.
3. **Optional configuration settings** → **Configuration values** (JSON), for example:

```json
{
  "env": {
    "ENABLE_PREFIX_DELEGATION": "true",
    "WARM_PREFIX_TARGET": "1"
  }
}
```

4. **Conflict resolution method**: choose **Overwrite** if you want EKS to replace your manual changes, or **Preserve** to keep them.
5. Save changes. Verify:

```bash
kubectl get pods -n kube-system -l k8s-app=aws-node
kubectl describe daemonset aws-node -n kube-system | grep -i prefix
```

To give the CNI its own permissions (best practice), choose a **Pod Identity** or **IRSA** role in the add-on's **IAM role** field. Create a role with `AmazonEKS_CNI_Policy`.

### 8.4 Amazon GuardDuty EKS add-on (Runtime Monitoring)

GuardDuty has two EKS features:

- **EKS Audit Log Monitoring**: reads control-plane audit logs (no agent).
- **EKS Runtime Monitoring**: uses an agent on each node (the add-on `aws-guardduty-agent`).

**Step A: Enable GuardDuty**

1. Console → **GuardDuty** → **Get started** → **Enable GuardDuty**.

**Step B: Enable Runtime Monitoring (automatic agent)**

1. GuardDuty → **Runtime Monitoring** → **Configuration** → **Edit**.
2. Runtime Monitoring: **Enable**.
3. **Automated agent configuration for Amazon EKS**: **Enable**.
4. Save.

GuardDuty then installs the agent as an EKS add-on by itself.

**Step C (alternative): install the add-on manually**

1. EKS → `my-eks` → Add-ons → Get more add-ons.
2. Tick **Amazon GuardDuty EKS Runtime Monitoring** (add-on name `aws-guardduty-agent`).
3. Create.

**Verify**

```bash
kubectl get pods -n amazon-guardduty
kubectl get daemonset -n amazon-guardduty
```

You should see one agent pod per node. Check **GuardDuty → Runtime Monitoring → Runtime coverage** to see the cluster as "Healthy".

> Nodes need network access to the GuardDuty endpoint. With automated agent configuration, GuardDuty creates the VPC endpoint for you.

### 8.5 Update or remove an add-on

- Add-on → **Edit** to change version or config.
- Add-on → **Remove**: keep "Preserve on cluster" if you only want EKS to stop managing it.

---

## Part 9: IRSA (IAM Roles for Service Accounts)

Goal: a pod in `default` namespace using service account `s3-reader-sa` can read S3, **without** node-level permissions.

### 9.1 Get the OIDC issuer URL

EKS → `my-eks` → **Overview** → copy **OpenID Connect provider URL**, like:

```
https://oidc.eks.ap-south-1.amazonaws.com/id/ABCDEF1234567890
```

The part after `/id/` is your `<OIDC_ID>`.

### 9.2 Create the IAM OIDC identity provider

1. IAM → **Identity providers** → **Add provider**.
2. Type: **OpenID Connect**.
3. Provider URL: paste the URL → **Get thumbprint**.
4. Audience: `sts.amazonaws.com`.
5. Add provider.

### 9.3 Create the permission policy

IAM → Policies → Create policy → JSON:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:ListAllMyBuckets", "s3:ListBucket", "s3:GetObject"],
      "Resource": "*"
    }
  ]
}
```

Name: `EksS3ReadPolicy`.

### 9.4 Create the IAM role with web identity trust

1. IAM → Roles → Create role → **Web identity**.
2. Identity provider: select the OIDC provider. Audience: `sts.amazonaws.com`. Next.
3. Attach `EksS3ReadPolicy`. Name: `eks-s3-reader-irsa-role`. Create.
4. Open the role → **Trust relationships** → **Edit trust policy**, and make it exactly like this (replace the placeholders):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/oidc.eks.ap-south-1.amazonaws.com/id/<OIDC_ID>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.ap-south-1.amazonaws.com/id/<OIDC_ID>:aud": "sts.amazonaws.com",
          "oidc.eks.ap-south-1.amazonaws.com/id/<OIDC_ID>:sub": "system:serviceaccount:default:s3-reader-sa"
        }
      }
    }
  ]
}
```

### 9.5 Create the Kubernetes service account and test pod

`irsa-test.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-reader-sa
  namespace: default
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/eks-s3-reader-irsa-role
---
apiVersion: v1
kind: Pod
metadata:
  name: irsa-test
  namespace: default
spec:
  serviceAccountName: s3-reader-sa
  containers:
    - name: aws-cli
      image: amazon/aws-cli:latest
      command: ["sleep", "3600"]
```

```bash
kubectl apply -f irsa-test.yaml
kubectl exec -it irsa-test -- aws sts get-caller-identity
kubectl exec -it irsa-test -- aws s3 ls
```

`get-caller-identity` should show the **assumed role** `eks-s3-reader-irsa-role`, not the node role.

Check the injected env vars:

```bash
kubectl exec irsa-test -- env | grep AWS
# AWS_ROLE_ARN, AWS_WEB_IDENTITY_TOKEN_FILE
```

---

## Part 10: EKS Pod Identity (the newer, simpler way)

Pod Identity needs the **EKS Pod Identity Agent** add-on (Part 5 step 6 or Part 8). No OIDC provider and no annotations are needed.

### 10.1 Confirm the agent

```bash
kubectl get pods -n kube-system | grep pod-identity
```

### 10.2 Create the IAM role

1. IAM → Roles → Create role → **Custom trust policy**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "pods.eks.amazonaws.com" },
      "Action": ["sts:AssumeRole", "sts:TagSession"]
    }
  ]
}
```

2. Attach `EksS3ReadPolicy`. Name: `eks-s3-reader-podidentity-role`.

### 10.3 Create the association in the console

1. EKS → `my-eks` → **Access** tab → **Pod Identity associations** → **Create**.
2. IAM role: `eks-s3-reader-podidentity-role`.
3. Kubernetes namespace: `default`.
4. Kubernetes service account: `pod-sa`.
5. Create.

### 10.4 Test

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pod-sa
  namespace: default
---
apiVersion: v1
kind: Pod
metadata:
  name: podid-test
  namespace: default
spec:
  serviceAccountName: pod-sa
  containers:
    - name: aws-cli
      image: amazon/aws-cli:latest
      command: ["sleep", "3600"]
```

```bash
kubectl apply -f podid-test.yaml
kubectl exec -it podid-test -- aws sts get-caller-identity
```

> Pods created **before** the association must be restarted to receive credentials.

### 10.5 IRSA vs Pod Identity

| | IRSA | Pod Identity |
|---|---|---|
| Needs OIDC provider | Yes | No |
| SA annotation | Required | Not required |
| Trust principal | OIDC provider | `pods.eks.amazonaws.com` |
| Role reuse across clusters | Edit trust policy per cluster | Same role works for many clusters |
| Needs agent add-on | No | Yes (Pod Identity Agent) |
| Works on Fargate | Yes | No |

---

## Part 11: Cordon, drain, uncordon

These are `kubectl` operations (the console does not do them).

```bash
kubectl get nodes

# Stop new pods from being scheduled on the node (existing pods keep running)
kubectl cordon <node-name>

# Evict pods safely, then you can patch / replace the node
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Bring it back
kubectl uncordon <node-name>
```

Cordoned nodes show `Ready,SchedulingDisabled`.

Typical flow when updating a node group: EKS cordons and drains old nodes automatically (that is what **Max unavailable** in the node group update config controls). Use manual drain when you debug or replace a single node.

Hint: if drain hangs, a PodDisruptionBudget or a bare pod (no controller) is usually the reason. Check:

```bash
kubectl get pdb -A
```

---

## Part 12: Auto scaling

### 12.1 Where the Auto Scaling Group is

A managed node group creates an ASG for you.

1. Console → **EC2** → left menu **Auto Scaling Groups**.
2. Find the one named `eks-ng-general-...`. Open it and look at **Capacity, Instance management, Activity, Tags**.

> Do **not** edit the ASG directly for managed node groups. Change min / desired / max in **EKS → Compute → Node group → Edit → Scaling configuration** instead.

### 12.2 Manual scaling

EKS → Cluster → Compute → select `ng-general` → **Edit** → set Desired size → Save.

### 12.3 Automatic node scaling with Cluster Autoscaler

Cluster Autoscaler watches for **Pending** pods and raises the ASG desired count. It also removes under-used nodes.

**Step 1: Tag the node group (so autodiscovery finds the ASG)**

EKS → Node group → Edit → **Kubernetes labels/tags** (tags section), add:

| Key | Value |
|---|---|
| `k8s.io/cluster-autoscaler/enabled` | `true` |
| `k8s.io/cluster-autoscaler/my-eks` | `owned` |

Then open the ASG in EC2 → **Tags** tab and confirm both tags exist there. If missing, add them on the ASG with **Tag new instances** on.

**Step 2: IAM policy**

IAM → Policies → Create policy → JSON, name `ClusterAutoscalerPolicy`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:DescribeAutoScalingInstances",
        "autoscaling:DescribeLaunchConfigurations",
        "autoscaling:DescribeScalingActivities",
        "autoscaling:DescribeTags",
        "ec2:DescribeImages",
        "ec2:DescribeInstanceTypes",
        "ec2:DescribeLaunchTemplateVersions",
        "ec2:GetInstanceTypesFromInstanceRequirements",
        "eks:DescribeNodegroup"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:SetDesiredCapacity",
        "autoscaling:TerminateInstanceInAutoScalingGroup"
      ],
      "Resource": "*"
    }
  ]
}
```

**Step 3: Role + Pod Identity association**

1. Create role `eks-cluster-autoscaler-role` with the Pod Identity trust policy (Part 10.2) and attach `ClusterAutoscalerPolicy`.
2. EKS → Access → Pod Identity associations → Create:
   - Role: `eks-cluster-autoscaler-role`
   - Namespace: `kube-system`
   - Service account: `cluster-autoscaler`

**Step 4: Install with Helm**

```bash
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm repo update

helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --set autoDiscovery.clusterName=my-eks \
  --set awsRegion=ap-south-1 \
  --set rbac.serviceAccount.create=true \
  --set rbac.serviceAccount.name=cluster-autoscaler

kubectl get pods -n kube-system | grep autoscaler
kubectl logs -n kube-system deploy/cluster-autoscaler-aws-cluster-autoscaler -f
```

> Use the Cluster Autoscaler image tag that matches your Kubernetes minor version (`--set image.tag=v1.XX.Y`).

**Step 5: Test scale up**

```bash
kubectl create deployment scale-test --image=nginx --replicas=1
kubectl set resources deployment scale-test --requests=cpu=500m,memory=256Mi
kubectl scale deployment scale-test --replicas=15

kubectl get pods            # some will be Pending
kubectl get nodes -w        # new nodes join after a few minutes (Ctrl+C to stop)
```

Scale down test:

```bash
kubectl scale deployment scale-test --replicas=0
```

After about 10 minutes of idle, the extra nodes are removed (down to the ASG minimum).

### 12.4 Pod-level scaling (HPA)

Install the **Metrics Server** add-on (Part 8), then:

```bash
kubectl autoscale deployment scale-test --cpu-percent=50 --min=2 --max=10
kubectl get hpa
```

HPA scales **pods**. Cluster Autoscaler scales **nodes**. Use both.

### 12.5 Modern alternative: Karpenter

Karpenter creates EC2 instances directly (no ASG), picks the best instance type, and is faster than Cluster Autoscaler. Learn it after you are comfortable with this guide: https://karpenter.sh

---

## Part 13: Troubleshooting cheat sheet

| Symptom | Likely cause | Fix |
|---|---|---|
| `error: You must be logged in to the server (Unauthorized)` | No access entry, or wrong AWS profile | Part 6.2, and `aws sts get-caller-identity` |
| `Unable to connect to the server: dial tcp ... i/o timeout` | Endpoint is private only, or your IP is blocked | Set endpoint to Public and private; check public access CIDRs |
| Nodes never appear | Node role missing policies, or private subnets have no NAT route | Check the 3 node policies and the NAT route table |
| Node `NotReady` | CNI problem or no IPs in subnet | `kubectl describe node`, `kubectl logs -n kube-system -l k8s-app=aws-node` |
| CoreDNS Pending | No nodes yet | Create the node group |
| Node group `Create failed` | Open **Health issues** on the node group tab | Fix the listed IAM or subnet issue |
| Pod cannot call AWS API | Trust policy `sub` mismatch (IRSA) or pod started before association (Pod Identity) | Re-check namespace and service account names, restart the pod |
| Pods Pending, "Too many pods" | Instance type ENI/IP limits | Bigger instances, or VPC CNI prefix delegation (Part 8.3) |
| Drain stuck | PDB or bare pod | `kubectl get pdb -A` |

Useful commands:

```bash
kubectl get pods -A
kubectl describe pod <pod>
kubectl logs <pod> --previous
kubectl get events -A --sort-by=.lastTimestamp
kubectl top nodes
```

---

## Part 14: Cleanup (do this to stop billing)

Delete in this order:

1. Kubernetes workloads that create AWS resources:
   ```bash
   kubectl delete svc --all-namespaces --field-selector metadata.name!=kubernetes
   kubectl delete ingress --all -A
   kubectl delete pvc --all -A
   ```
   (Load balancers and EBS volumes must be gone, or the VPC delete will fail.)
2. `helm uninstall cluster-autoscaler -n kube-system`
3. EKS → Cluster → Compute → **Delete node group**.
4. EKS → Add-ons → remove extra add-ons (optional, cluster deletion removes them).
5. EKS → **Delete cluster**.
6. VPC → delete the **NAT Gateway** → release the **Elastic IP** → delete the VPC (`eks-vpc`).
7. IAM → delete roles: `eksClusterRole`, `eksNodeRole`, IRSA, Pod Identity and autoscaler roles, plus the OIDC provider and custom policies.
8. GuardDuty → disable if you do not want its charges.

Then check **Billing → Cost Explorer** the next day to confirm nothing is still running.

---

## Practice checklist

- [ ] `aws configure --profile eks-lab` saved, `get-caller-identity` works
- [ ] Cluster roles, node role, VPC created
- [ ] Cluster created, `kubectl get svc` works from terminal
- [ ] Node group created, 2 nodes `Ready`
- [ ] VPC CNI add-on edited with a config value
- [ ] GuardDuty runtime agent pods running
- [ ] IRSA pod assumes the role
- [ ] Pod Identity pod assumes the role
- [ ] Cordon, drain, uncordon done on a node
- [ ] Cluster Autoscaler scaled nodes up and down
- [ ] Cleanup finished
