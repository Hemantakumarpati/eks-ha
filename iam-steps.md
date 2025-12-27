# Step-by-Step IAM Configuration Guide for Production EKS

This guide provides a comprehensive, step-by-step walk-through for configuring the Identity and Access Management (IAM) requirements for the Production EKS cluster, with a focus on high availability and secure GitHub Actions automation via OIDC.

## 1. Overview of IAM Architecture

The architecture relies on the principle of **Least Privilege** and uses **federated identity** for CI/CD.

- **EKS Cluster Role**: Allows the EKS control plane to manage AWS resources (like ELBs and EBS volumes) on your behalf.
- **EKS Node Group Role**: Allows worker nodes to connect to the cluster, pull container images, and send logs to CloudWatch.
- **GitHub OIDC Role**: A federated role that GitHub Actions assumes to deploy infrastructure without needing long-lived access keys.

---

## 2. Step-by-Step Configuration

### Step 1: Deploy the OIDC Identity Provider
To allow GitHub Actions to talk to AWS securely, you must establish trust between them.

1. **Option A (CloudFormation - Recommended)**:
   The `iam.yaml` template already includes the `GitHubOIDCProvider` resource. Deploying this stack automatically handles the setup.
   
2. **Option B (Manual Console setup)**:
   - Go to **IAM Console** > **Identity Providers** > **Add Provider**.
   - **Provider Type**: OpenID Connect.
   - **Provider URL**: `https://token.actions.githubusercontent.com`.
   - **Audience**: `sts.amazonaws.com`.
   - Click **Get Thumbprint**.

### Step 2: Configure the Cluster Role
1. Create a role for the **EKS - Cluster** service.
2. Attach the managed policy: `AmazonEKSClusterPolicy`.
3. (Optional) If using VPC CNI for HA networking across 3 AZs, ensure the role has permissions to manage network interfaces.

### Step 3: Configure the Node Group Role
Worker nodes need permissions to function within the AWS ecosystem.
1. Create a role for the **EC2** service.
2. Attach the following managed policies:
   - `AmazonEKSWorkerNodePolicy`
   - `AmazonEKS_CNI_Policy`
   - `AmazonEC2ContainerRegistryReadOnly`
   - `CloudWatchAgentServerPolicy` (for logging/monitoring).

### Step 4: Configure the GitHub Actions OIDC Role
This is the role your CI/CD pipeline assumes.

1. **Trust Policy**: The role must trust the GitHub OIDC provider.
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Principal": {
           "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
         },
         "Action": "sts:AssumeRoleWithWebIdentity",
         "Condition": {
           "StringLike": {
             "token.actions.githubusercontent.com:sub": "repo:<GITHUB_ORG>/<GITHUB_REPO>:*"
           }
         }
       }
     ]
   }
   ```
2. **Permissions**: Attach `AdministratorAccess` (for infrastructure provisioning) or a custom restricted policy.

---

## 3. Integrating with Kubernetes (IRSA)

For a production cluster, you should also set up **IAM Roles for Service Accounts (IRSA)**. This allows your Java application pods running in EKS to have their own IAM roles (e.g., to access S3 or Secrets Manager) instead of sharing the Node Group's role.

1. **Enable OIDC on Cluster**: This is done automatically in our `eks-cluster.yaml`.
2. **Create IAM Role for Pods**: A role with a trust policy that trusts the **Cluster's OIDC issuer URL**.
3. **Annotate Service Account**:
   ```yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: my-app-sa
     annotations:
       eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/my-app-pod-role
   ```

---

## 4. Security Checklist

- [ ] **No User Access Keys**: Ensure no IAM Users are used in GitHub Actions; stay with OIDC.
- [ ] **Thumbprint Validation**: Periodically verify the GitHub OIDC thumbprint.
- [ ] **Role Scoping**: Ensure the GitHub OIDC role is restricted to your specific repository using the `Condition` block.
- [ ] **Logging**: Enable IAM Access Advisor to monitor which permissions are actually being used.

---

## 5. Troubleshooting: "Could not assume role with OIDC"

If you see the error `Not authorized to perform sts:AssumeRoleWithWebIdentity`, check these 3 things:

### 1. Exactly Match the Trust Policy
The most common cause is a mismatch between your GitHub Repo path and the IAM Trust Policy. 
- Go to **IAM** > **Roles** > Your Role > **Trust relationships**.
- Click **Edit trust policy**.
- Ensure the `sub` filter exactly matches your repository:
  ```json
  "Condition": {
    "StringLike": {
      "token.actions.githubusercontent.com:sub": "repo:Hemantakumarpati/eks-ha:*"
    }
  }
  ```
  *(Change `Hemantakumarpati/eks-ha` if your user name is different!)*

### 1. "GitHubOIDCProvider" Conflict Resolution
We have updated the templates to use your **existing** OIDC Provider instead of creating a new one.

**Action Required**:
- When triggering the GitHub Action, you **must** provide your existing OIDC Provider ARN in the `oidc_arn` input.
- You can find this in **IAM Console** > **Identity Providers** > Click on `token.actions.githubusercontent.com` > Copy the **ARN**.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringLike": {
                    "token.actions.githubusercontent.com:sub": "repo:Hemantakumarpati/eks-ha:*"
                },
                "StringEquals": {
                    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
                }
            }
        }
    ]
}
```
*(Make sure `Hemantakumarpati` and `eks-ha` match the exact case visible in your GitHub URL).*

### 2. Verify the Identity Provider Thumbprints
GitHub rotated their certificates. Ensure your **Identity Provider** in IAM has **both** of these thumbprints to be safe:
1. `6938fd4d98bab03faadb97b34396831e3780aea1`
2. `1c58a369d786a9840c4416112ed24b8df25920d3` 

### 3. Check the "role-to-assume" Secret
- Go to GitHub Secrets.
- Click **Edit** on `AWS_ROLE_ARN`.
- Ensure there are **no spaces** at the beginning or end of the ARN.
- Example: `arn:aws:iam::123456789012:role/GitHubActionsOIDCRole`

### 4. Check the "permissions" in deploy.yml
Ensure your workflow still has this at the top (required for OIDC):
```yaml
permissions:
  id-token: write
  contents: read
```
