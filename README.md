# IDP Guardrails — OPA/Gatekeeper Admission Policies

Kubernetes admission policies for enforcing governance on ACK-provisioned AWS resources.
These policies run as admission webhooks — resources violating a policy are **rejected before they reach the ACK controller**, and therefore never reach AWS.

## Policies

| Policy | File | What it enforces |
|---|---|---|
| **Required Labels** | `policies/required-labels/` | All ACK resources must have `team`, `environment`, `cost-center` labels |
| **Approved RDS Instance Types** | `policies/allowed-rds-instance-types/` | Only approved RDS instance sizes (blocks expensive types) |
| **S3 Encryption Required** | `policies/s3-encryption-required/` | All S3 buckets must have KMS encryption configured |

## Structure

```
idp-guardrails/
├── templates/           ← ConstraintTemplate CRDs (Rego policy definitions)
│   ├── required-labels-template.yaml
│   ├── allowed-rds-instance-types-template.yaml
│   └── s3-encryption-required-template.yaml
└── constraints/         ← Constraint CRDs (enforcement rules + parameters)
    ├── ack-required-labels.yaml
    ├── rds-approved-instance-types.yaml
    └── s3-encryption-required.yaml
```

## Deploy

```bash
# Install Gatekeeper (if not already installed)
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
helm install gatekeeper gatekeeper/gatekeeper \
  --namespace gatekeeper-system --create-namespace

# Apply all policies
kubectl apply -f templates/
kubectl apply -f constraints/

# Verify
kubectl get constrainttemplates
kubectl get constraints
```

## Test

```bash
# Test 1: Create ACK S3 bucket without required labels → should be REJECTED
kubectl apply -f - <<EOF
apiVersion: s3.services.k8s.aws/v1alpha1
kind: Bucket
metadata:
  name: test-no-labels
  namespace: default
spec:
  name: test-no-labels-bucket
EOF

# Test 2: Create RDS with disallowed instance type → should be REJECTED
kubectl apply -f - <<EOF
apiVersion: rds.services.k8s.aws/v1alpha1
kind: DBInstance
metadata:
  name: test-expensive-db
  namespace: default
  labels:
    team: test
    environment: development
    cost-center: test-001
spec:
  dbInstanceIdentifier: test-expensive-db
  dbInstanceClass: db.r6g.xlarge   # ← not in approved list
  engine: postgres
EOF
```
