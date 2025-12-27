# Production EKS Cluster Implementation Plan (HA Upgrade)

This plan upgrades the existing architecture to a **truly production-ready, high-availability EKS cluster**. The design expands from a 2-AZ demo setup to a 3-AZ resilient architecture, eliminating single points of failure.

## User Review Required

> [!IMPORTANT]
> **Cost vs Availability**: To achieve true production HA, we will use **3 NAT Gateways** (one per AZ). This ensures that if one AZ fails, the other two remain fully operational with outbound connectivity. This will increase AWS costs compared to the previous single-NAT setup.

> [!IMPORTANT]
> **Multi-Zone Strategy**: We will configure the Managed Node Group to spread across **3 Availability Zones** with a minimum of **3 nodes** (one per zone).

## Proposed Changes

### [Component] Networking (VPC)
#### [MODIFY] [vpc.yaml](file:///c:/Users/heman/learning/aws/native-eks/templates/vpc.yaml)
- Expand to **3 Availability Zones**.
- 3 Public Subnets (CIDRs: 10.0.0.0/24, 10.0.1.0/24, 10.0.2.0/24).
- 3 Private Subnets (CIDRs: 10.0.10.0/24, 10.0.11.0/24, 10.0.12.0/24).
- **3 NAT Gateways**: Each public subnet will host a NAT GW to serve its corresponding private subnet.
- 3 Private Route Tables (one per private subnet pointing to its local NAT GW).

---

### [Component] EKS Control Plane
#### [MODIFY] [eks-cluster.yaml](file:///c:/Users/heman/learning/aws/native-eks/templates/eks-cluster.yaml)
- Update `SubnetIds` to include all **3 Private Subnets**.
- Ensure `EndpointPrivateAccess` is true for internal cluster communication.

---

### [Component] Compute (Node Groups)
#### [MODIFY] [node-group.yaml](file:///c:/Users/heman/learning/aws/native-eks/templates/node-group.yaml)
- **Desired Capacity: 3** (One node per AZ).
- **Min Size: 3**, **Max Size: 6**.
- Ensure `Subnets` include all **3 Private Subnets**.

---

### [Component] Orchestration
#### [MODIFY] [master.yaml](file:///c:/Users/heman/learning/aws/native-eks/templates/master.yaml)
- Pass 3 Subnet IDs from `NetworkStack` to `EKSStack` and `NodeGroupStack`.

## Verification Plan

### Automated Verification
- **Validate CloudFormation**: `aws cloudformation validate-template --template-body file://templates/master.yaml`
- **Linting**: Run `cfn-lint` on updated templates.

### Manual Verification
1. **AZ Distribution**: After deployment, run `kubectl get nodes -L topology.kubernetes.io/zone` to verify nodes are in different AZs.
2. **NAT Gateway Test**: Terminate a NAT Gateway and verify that pods in other AZs still have internet access.
3. **Connectivity**: Run a test pod in each subnet to verify outbound internet access through NAT Gateways.

---

## Bootstrap & GitHub Actions Setup

To enable automated deployments via OIDC, follow these setup steps.

### Step 1: Manual Bootstrap (First Deployment)
Since GitHub Actions needs the IAM Role created by the stack, you must run the first deployment manually from your local machine (ensure your local CLI is set to `ap-south-1`):

```powershell
./deploy.sh <YOUR_S3_BUCKET_NAME> eks-production-ha <YOUR_AWS_CONNECTION_ARN> Hemantakumarpati/eks-ha
```

### Step 2: Configure GitHub Secrets
Once the initial deployment succeeds, go to your GitHub Repo **Settings > Secrets and variables > Actions** and add the following secrets:

| Secret Name | Description | Where to find it? |
| :--- | :--- | :--- |
| **`AWS_ROLE_ARN`** | The IAM Role for GitHub Actions | CloudFormation > `eks-production-ha` > **Outputs** > `GitHubRoleArn` |
| **`AWS_CONNECTION_ARN`** | Your AWS CodeConnections ARN | AWS Console > CodeConnections > Connections |
| **`S3_BUCKET`** | The S3 bucket for templates | AWS Console > S3 |

### Step 3: Manual Deployment
After adding the secrets, you can manually trigger the deployment:
1. Go to your GitHub Repository **Actions** tab.
2. Select the **"Deploy Production EKS (HA)"** workflow.
3. Click **Run workflow** and select the `main` branch.

