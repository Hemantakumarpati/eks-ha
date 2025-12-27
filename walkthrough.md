# EKS High Availability Architecture & Production Walkthrough

This document showcases a **production-grade, multi-zone EKS cluster** designed for maximum resilience and high availability.

## 1. High Availability Strategy: The 3-AZ Model

To meet production standards, we have moved beyond a simple multi-AZ setup to a **3-Availability Zone (3-AZ) Architecture**. This ensures that even if an entire AWS Data Center (AZ) goes offline, the cluster retains 66% of its capacity, preventing downtime.

### Key HA Components:
1.  **Triple-AZ Subnets**: 3 Public and 3 Private subnets spread across 3 unique Availability Zones.
2.  **Redundant NAT Gateways**: **3 NAT Gateways** (one per AZ). Unlike a single NAT gateway setup, this prevents a single AZ failure from cutting off internet access for the entire cluster.
3.  **Cross-Zone Node Scaling**: The Managed Node Group is configured with a **Desired Capacity of 3**, ensuring at least one node is always running in each AZ.

---

## Visualizing the HA Architecture

```mermaid
graph TD
    subgraph "VPC (10.0.0.0/16)"
        subgraph "AZ-1"
            Pub1[Public Subnet 1] --> NAT1[NAT Gateway 1]
            Priv1[Private Subnet 1] --> Node1[EKS Node 1]
            Node1 --> NAT1
        end
        subgraph "AZ-2"
            Pub2[Public Subnet 2] --> NAT2[NAT Gateway 2]
            Priv2[Private Subnet 2] --> Node2[EKS Node 2]
            Node2 --> NAT2
        end
        subgraph "AZ-3"
            Pub3[Public Subnet 3] --> NAT3[NAT Gateway 3]
            Priv3[Private Subnet 3] --> Node3[EKS Node 3]
            Node3 --> NAT3
        end
        
        NAT1 --> IGW[Internet Gateway]
        NAT2 --> IGW
        NAT3 --> IGW
    end

    ControlPlane[EKS Control Plane] -.-> Node1
    ControlPlane -.-> Node2
    ControlPlane -.-> Node3
```

## 2. Updated CloudFormation Highlights

### VPC Level ([vpc.yaml](file:///c:/Users/heman/learning/aws/native-eks/templates/vpc.yaml))
-   **CIDRs**: Carefully partitioned to avoid overlap (10.0.0.0/24, 10.0.1.0/24, 10.0.2.0/24 for Public; 10.0.10.0/24, 10.0.11.0/24, 10.0.12.0/24 for Private).
-   **Route Isolation**: Each private subnet has its own route table pointing to its local NAT Gateway, ensuring AZ independence.

### Compute Level ([node-group.yaml](file:///c:/Users/heman/learning/aws/native-eks/templates/node-group.yaml))
-   **Scaling**: `Min: 3`, `Desired: 3`, `Max: 6`.
-   **Distribution**: AWS EKS automatically manages the distribution of nodes across the provided subnets to maintain balance.

## 3. Production Readiness Checklist

| Feature | Status | Description |
| :--- | :--- | :--- |
| **Multi-AZ Control Plane** | ✅ | Managed by AWS EKS. |
| **Multi-AZ Worker Nodes** | ✅ | Spread across 3 AZs. |
| **Outbound HA** | ✅ | 3 NAT Gateways for redundant internet egress. |
| **Scalability** | ✅ | Managed Node Group with Auto-scaling (3 to 6 nodes). |
| **Modular IaC** | ✅ | Nested CloudFormation stacks for ease of management. |

---

## Deployment & CI/CD Strategy

This cluster is managed via **GitHub Actions** using a secure, keyless OIDC integration.

1.  **Environment**: Provisioned in **`ap-south-1` (Mumbai)**.
2.  **Trigger**: Manual trigger via `workflow_dispatch` for controlled infrastructure updates.
3.  **Security**: No permanent AWS Access Keys; utilizes short-lived OIDC tokens.

---

## Post-Deployment Verification

Now that the cluster is LIVE, follow these steps to verify it from your local machine:

### 1. Update your local Kubeconfig
```bash
aws eks update-kubeconfig --name production-eks-ha --region ap-south-1
```

### 2. Verify Node Health
```bash
kubectl get nodes -o wide
```
*You should see 3 nodes in `Ready` status, distributed across different subnets/AZs.*

### 3. Check System Pods
```bash
kubectl get pods -n kube-system
```

---

## What's Next?
- **Application Deployment**: Use the pre-configured `cicd-pipeline.yaml` to deploy your Java services.
- **Monitoring**: Consider enabling Container Insights or installing Prometheus/Grafana for deep visibility.
- **Security**: Implement Network Policies to restrict pod-to-pod communication.
