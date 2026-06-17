# EKS-kubectl-agent Platform: Architecture Specification

## Complete System Overview

```
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                     EKS-kubectl-agent Platform Architecture                                                  │
├─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                                               │
│  ┌──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐   │
│  │ LAYER 1: USER & FRONTEND                                                                                             │   │
│  ├──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤   │
│  │                                                                                                                       │   │
│  │  [👤 Users: Admin, Developers, SRE]  ──────►  [Browser]  ──────►  [CloudFront CDN]                                 │   │
│  │                                                                      │                                                │   │
│  │                                                                      ├─► S3 Static Export (Next.js)                 │   │
│  │                                                                      ├─► Cognito Auth (JWT validation)              │   │
│  │                                                                      └─► Dashboard UI                               │   │
│  │                                                                                                                       │   │
│  └──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘   │
│                                                         ║ HTTPS Request with JWT                                            │
│                                                         ▼                                                                    │
│  ┌──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐   │
│  │ LAYER 2: API GATEWAY & REQUEST HANDLING                                                                            │   │
│  ├──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤   │
│  │                                                                                                                       │   │
│  │  [API Gateway HTTP/v2]  (Request validation, auth, rate limiting)                                                   │   │
│  │     ├─ POST /api/v1/diagnostics         (diagnose-start lambda)                                                     │   │
│  │     ├─ POST /api/v1/compliance          (compliance-check lambda)                                                   │   │
│  │     ├─ POST /api/v1/topology            (cluster-topology lambda)                                                   │   │
│  │     ├─ GET  /api/v1/accounts            (list-accounts lambda)                                                      │   │
│  │     ├─ GET  /api/v1/audit               (audit-logs lambda)                                                         │   │
│  │     ├─ POST /api/v1/diagnostics/root-cause         (NEW: root-cause lambda)                                        │   │
│  │     ├─ GET  /api/v1/incidents/{id}/timeline        (NEW: timeline lambda)                                          │   │
│  │     ├─ POST /api/v1/runbooks/{id}/execute          (NEW: runbook-executor lambda)                                  │   │
│  │     ├─ GET  /api/v1/slo                            (NEW: slo-engine lambda)                                         │   │
│  │     └─ POST /api/v1/incidents/{id}/auto-summary    (NEW: summary-generator lambda)                                 │   │
│  │                                                                                                                       │   │
│  │  [JWT Authorizer]  (validate Cognito token, extract permissions)                                                    │   │
│  │  [IAM Auth]        (service-to-service calls)                                                                        │   │
│  │                                                                                                                       │   │
│  └──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘   │
│                          ║ Lambda Invocation with context (cluster, namespace, action)                                      │
│                          ▼                                                                                                    │
│  ┌──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐   │
│  │ LAYER 3: CORE SERVICE LAMBDAS (Read-Only Diagnostics & Data Collection)                                             │   │
│  ├──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤   │
│  │                                                                                                                       │   │
│  │  ┌─────────────────────────┐  ┌──────────────────────┐  ┌─────────────────────┐  ┌──────────────────────┐           │   │
│  │  │ Diagnostics Lambda      │  │ Compliance Lambda    │  │ Topology Lambda     │  │ List Accounts Lam   │           │   │
│  │  ├─────────────────────────┤  ├──────────────────────┤  ├─────────────────────┤  ├──────────────────────┤           │   │
│  │  │ • Query pod status      │  │ • Run policy checks  │  │ • Collect resources │  │ • List assigned acc │           │   │
│  │  │ • Node metrics          │  │ • Security baseline  │  │ • Node utilization  │  │ • Filter by role    │           │   │
│  │  │ • Event logs            │  │ • Compliance scoring │  │ • Service endpoints │  │ • Return account ID │           │   │
│  │  │ • Storage info          │  │ • Store results      │  │ • Network policies  │  │                      │           │   │
│  │  │ • RBAC validation       │  │                      │  │ • Health analysis   │  │                      │           │   │
│  │  └─────────────────────────┘  └──────────────────────┘  └─────────────────────┘  └──────────────────────┘           │   │
│  │                                                                                                                       │   │
│  └──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘   │
│                                       ║ Event/Result → Store + Process                                                      │
│                                       ▼                                                                                      │
│  ┌──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐   │
│  │ LAYER 4: AI & ANALYTICS SERVICES (Intelligent Processing)                                                           │   │
│  ├──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤   │
│  │                                                                                                                       │   │
│  │  ┌──────────────────────┐  ┌──────────────────────┐  ┌──────────────────────┐  ┌──────────────────────┐             │   │
│  │  │ Root Cause Engine    │  │ Incident Timeline    │  │ Runbook Orchestrator │  │ SLO Engine           │             │   │
│  │  ├──────────────────────┤  ├──────────────────────┤  ├──────────────────────┤  ├──────────────────────┤             │   │
│  │  │ • Hypothesis scoring │  │ • Event aggregation  │  │ • Action registry    │  │ • SLO definitions    │             │   │
│  │  │ • Confidence ranks   │  │ • Temporal ordering  │  │ • Dry-run execution  │  │ • Burn calculations  │             │   │
│  │  │ • Evidence linking   │  │ • Change context     │  │ • Approval workflow  │  │ • Alert thresholds   │             │   │
│  │  │ • Suggested actions  │  │ • Root cause hints   │  │ • Execution logs     │  │ • Trend tracking     │             │   │
│  │  └──────────────────────┘  └──────────────────────┘  └──────────────────────┘  └──────────────────────┘             │   │
│  │                                                                                                                       │   │
│  │  ┌──────────────────────┐  ┌──────────────────────┐  ┌──────────────────────┐                                       │   │
│  │  │ Change Correlator    │  │ Summary Generator    │  │ Scorer/Ranker        │                                       │   │
│  │  ├──────────────────────┤  ├──────────────────────┤  ├──────────────────────┤                                       │   │
│  │  │ • Deployment changes │  │ • Timeline narrative │  │ • Predict severity   │                                       │   │
│  │  │ • Config diffs       │  │ • Root cause link    │  │ • Rank by impact     │                                       │   │
│  │  │ • IAM changes        │  │ • Action summary     │  │ • ML feedback loop   │                                       │   │
│  │  │ • Temporal proximity │  │ • Export formats     │  │ • Tune thresholds    │                                       │   │
│  │  └──────────────────────┘  └──────────────────────┘  └──────────────────────┘                                       │   │
│  │                                                                                                                       │   │
│  └──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘   │
│                              ║ Store results + Create incidents                                                             │
│                              ▼                                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐   │
│  │ LAYER 5: PERSISTENT STATE & OBSERVABILITY                                                                            │   │
│  ├──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤   │
│  │                                                                                                                       │   │
│  │  ┌────────────────────────────┐  ┌─────────────────────┐  ┌──────────────────────┐                                  │   │
│  │  │ DynamoDB Tables            │  │ CloudWatch          │  │ AWS X-Ray (APM)      │                                  │   │
│  │  ├────────────────────────────┤  ├─────────────────────┤  ├──────────────────────┤                                  │   │
│  │  │ incidents                  │  │ Lambda logs         │  │ Distributed traces   │                                  │   │
│  │  │ timeline_events            │  │ Custom metrics      │  │ Service dependency   │                                  │   │
│  │  │ runbook_jobs               │  │ Alarms & thresholds │  │ Latency percentiles  │                                  │   │
│  │  │ slo_configurations         │  │ Log insights        │  │ Error tracking       │                                  │   │
│  │  │ audit_logs                 │  │ Dashboards          │  │ Performance analysis │                                  │   │
│  │  │ cluster_assignments        │  │                     │  │                      │                                  │   │
│  │  │ compliance_snapshots       │  │                     │  │                      │                                  │   │
│  │  └────────────────────────────┘  └─────────────────────┘  └──────────────────────┘                                  │   │
│  │                                                                                                                       │   │
│  │  All Lambda functions emit OpenTelemetry spans to X-Ray and metrics to CloudWatch Logs                              │   │
│  │                                                                                                                       │   │
│  └──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘   │
│                                ║ Assume spoke role with STS                                                                 │
│                                ▼                                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐   │
│  │ LAYER 6: IAM & MULTI-ACCOUNT ACCESS                                                                                 │   │
│  ├──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤   │
│  │                                                                                                                       │   │
│  │  Central Account (307946672793)                                                                                     │   │
│  │     ├─ Lambda Execution Role (assume-role permission)                                                               │   │
│  │     ├─ API Gateway                                                                                                  │   │
│  │     ├─ Cognito User Pool                                                                                            │   │
│  │     └─ DynamoDB, CloudWatch, X-Ray (shared)                                                                         │   │
│  │                                                                                                                       │   │
│  │  Spoke Roles (Member Accounts)                                                                                      │   │
│  │     ├─ STS Assume Role Policy → eks-kubectl-agent-dev-lambda-role                                                   │   │
│  │     ├─ eks:DescribeCluster, eks:ListClusters                                                                        │   │
│  │     ├─ EKS Access Entries (Cognito<-->EKS mapping)                                                                  │   │
│  │     └─ ec2:Describe* (nodes, security groups, etc)                                                                  │   │
│  │                                                                                                                       │   │
│  │  Cross-Account Access Pattern:                                                                                       │   │
│  │     User JWT (Cognito) → Lambda (hub) → assume STS role (spoke) → query EKS API (member account)                   │   │
│  │                                                                                                                       │   │
│  └──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘   │
│                          ║ Kubernetes API queries with kubeconfig                                                           │
│                          ▼                                                                                                   │
│  ┌──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐   │
│  │ LAYER 7: EKS CLUSTERS & KUBERNETES (Multi-Region, Multi-Account)                                                    │   │
│  ├──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤   │
│  │                                                                                                                       │   │
│  │  ┌────────────────────────────────┐  ┌────────────────────────────────┐  ┌──────────────────────────────────────┐  │   │
│  │  │ EKS Cluster 1 (Account N)      │  │ EKS Cluster 2 (Account N)      │  │ ... EKS Cluster M                    │  │   │
│  │  ├────────────────────────────────┤  ├────────────────────────────────┤  ├──────────────────────────────────────┤  │   │
│  │  │                                │  │                                │  │                                      │  │   │
│  │  │ Control Plane                  │  │ Control Plane                  │  │ Control Plane                        │  │   │
│  │  │  • Kubernetes API Server       │  │  • Kubernetes API Server       │  │  • Kubernetes API Server            │  │   │
│  │  │  • etcd                        │  │  • etcd                        │  │  • etcd                             │  │   │
│  │  │                                │  │                                │  │                                      │  │   │
│  │  │ Worker Nodes + Pods            │  │ Worker Nodes + Pods            │  │ Worker Nodes + Pods                 │  │   │
│  │  │  • metrics-server              │  │  • metrics-server              │  │  • metrics-server                   │  │   │
│  │  │  • kubelet (stats)             │  │  • kubelet (stats)             │  │  • kubelet (stats)                  │  │   │
│  │  │  • Container Insights (opt)    │  │  • Container Insights (opt)    │  │  • Container Insights (opt)         │  │   │
│  │  │  • Network Policies            │  │  • Network Policies            │  │  • Network Policies                 │  │   │
│  │  │  • RBAC (service accounts)     │  │  • RBAC (service accounts)     │  │  • RBAC (service accounts)          │  │   │
│  │  │                                │  │                                │  │                                      │  │   │
│  │  └────────────────────────────────┘  └────────────────────────────────┘  └──────────────────────────────────────┘  │   │
│  │                                                                                                                       │   │
│  │  Cross-Cluster Connectivity:                                                                                         │   │
│  │     • VPC Peering (for same region)                                                                                  │   │
│  │     • Transit Gateway (for multi-region)                                                                             │   │
│  │     • EKS Access Entries (for user RBAC)                                                                             │   │
│  │                                                                                                                       │   │
│  └──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                                                               │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Data Flow: From Event Detection to Resolution

```
INCIDENT LIFECYCLE
═══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════

┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                                   PHASE 1: DETECTION                                                         │
├─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                                               │
│   Event occurs in cluster: Pod crash, node pressure, network disconnection, deployment failure, etc.                       │
│                                ↓                                                                                             │
│   User clicks "Diagnose" or API calls POST /api/v1/diagnostics                                                             │
│                                ↓                                                                                             │
│   API Gateway → JWT Auth → diagnostics-start Lambda                                                                         │
│                                ↓                                                                                             │
│   Create async job in DynamoDB (job_id, status=QUEUED)                                                                    │
│                                ↓                                                                                             │
│   Return job_id to frontend; start async diagnostics-worker                                                                │
│                                                                                                                               │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                           PHASE 2: DATA COLLECTION (Worker Lambda)                                          │
├─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                                               │
│   diagnostics-worker processes job in parallel:                                                                             │
│                                                                                                                               │
│   1. Assume spoke role (STS) → get EKS kubeconfig                                                                           │
│   2. Query Kubernetes API:                                                                                                  │
│       • Pod status (all namespaces): restarts, state, events, containers                                                    │
│       • Node conditions: CPU, memory, disk, PID pressure, cordoned                                                          │
│       • Events: warnings, errors, timeline                                                                                 │
│       • Storage: PVC, PV, quota usage                                                                                      │
│       • RBAC: service accounts, role bindings                                                                               │
│       • Network: services, endpoints, network policies, ingresses                                                            │
│   3. Gather metrics (via metrics-server or kubelet fallback)                                                                │
│   4. Query CloudWatch for Container Insights (if enabled)                                                                   │
│   5. Emit OpenTelemetry spans to X-Ray (trace entire operation)                                                             │
│                                                                                                                               │
│   Update DynamoDB: job_id → status=PROCESSING, collected_data                                                             │
│   Log to CloudWatch Logs (indexed for searching)                                                                            │
│                                                                                                                               │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                           PHASE 3: ANALYSIS & ROOT CAUSE                                                    │
├─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                                               │
│   Root Cause Engine Lambda invoked on completed diagnostics                                                                 │
│                                                                                                                               │
│   1. Analyze collected data:                                                                                                │
│       • Pod restarts → check logs for OOMKilled, CrashLoopBackOff, ImagePullBackOff                                        │
│       • Node pressure → correlate with pod evictions + storage issues                                                       │
│       • Event timeline → identify trigger event (deploy, config change, node failure)                                       │
│       • Network errors → check network policies + DNS + service endpoints                                                   │
│   2. Score hypotheses:                                                                                                       │
│       • Most likely: out of memory (confidence: 95%)                                                                         │
│       • Secondary: resource quota exceeded (confidence: 80%)                                                                 │
│       • Tertiary: container health check failed (confidence: 60%)                                                            │
│   3. Fetch evidence from timeline:                                                                                           │
│       • Pod events: "MemoryPressure" observed at T-5min                                                                     │
│       • Node events: "EvictedPods" at T-2min                                                                                │
│   4. Link to compliance/security context if relevant                                                                        │
│                                                                                                                               │
│   Store root_cause_analysis in DynamoDB → incidents table                                                                  │
│                                                                                                                               │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                            PHASE 4: TIMELINE & CONTEXT                                                      │
├─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                                               │
│   Incident Timeline Lambda builds narrative:                                                                                │
│                                                                                                                               │
│   1. Aggregate events from multiple sources:                                                                                │
│       • Kubernetes events (pod, node, event objects)                                                                        │
│       • Compliance scan results (if recent)                                                                                 │
│       • Audit logs (deployments, RBAC changes, user actions)                                                                │
│       • Diagnostics runs (past results if recurring issue)                                                                  │
│   2. Order chronologically:                                                                                                 │
│       T-60min: New deployment rolled out (v2.1 → v2.2)                                                                     │
│       T-45min: Pod memory usage gradually increased                                                                         │
│       T-20min: First OOMKilled pod observed                                                                                 │
│       T-10min: Node enters memory pressure                                                                                  │
│       T-0min:   Pod eviction cascade triggered                                                                              │
│   3. Link to probable cause: "Memory leak introduced in v2.2 deployment"                                                    │
│                                                                                                                               │
│   Store timeline events in DynamoDB → timeline_events table (TTL=90 days)                                                  │
│                                                                                                                               │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                          PHASE 5: CHANGE CORRELATION                                                        │
├─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                                               │
│   Change Correlator Lambda queries audit logs:                                                                              │
│                                                                                                                               │
│   1. Recent deployments (past 2 hours):                                                                                     │
│       • Which service? deployment/nginx-app                                                                                 │
│       • What changed? image: v2.1 → v2.2                                                                                   │
│       • When? 60 minutes ago, by user@example.com                                                                           │
│       • Confidence: HIGH (deployment correlates with memory increase)                                                       │
│   2. Configuration changes:                                                                                                 │
│       • Increased resource limits? No                                                                                       │
│       • Quota changes? No                                                                                                   │
│   3. IAM/RBAC changes:                                                                                                      │
│       • New service account? No                                                                                             │
│       • Permission changes? No                                                                                              │
│                                                                                                                               │
│   Correlation result: "Most likely trigger = nginx-app deployment v2.2 (60 min ago)"                                       │
│                                                                                                                               │
│   Store in incidents table → change_correlation field                                                                       │
│                                                                                                                               │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                            PHASE 6: REMEDIATION MATCHING                                                    │
├─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                                               │
│   Runbook Orchestrator Lambda finds applicable actions:                                                                     │
│                                                                                                                               │
│   For "out of memory" diagnosis:                                                                                            │
│                                                                                                                               │
│   Runbook Option 1 (Immediate relief):                                                                                     │
│     • Name: "Scale Up Deployment Replicas"                                                                                 │
│     • Prerequisites: deployment exists, has replicas                                                                        │
│     • Action: kubectl scale deployment/nginx-app --replicas=5                                                               │
│     • Blast radius: Medium (more pods = more resource contention initially)                                                │
│     • Rollback: Easy (scale back down)                                                                                      │
│     • Dry-run: ✓ supported                                                                                                  │
│                                                                                                                               │
│   Runbook Option 2 (Root fix):                                                                                              │
│     • Name: "Rollback Deployment to v2.1"                                                                                  │
│     • Prerequisites: deployment has revision history                                                                        │
│     • Action: kubectl rollout undo deployment/nginx-app                                                                     │
│     • Blast radius: Low (reverting to known-good version)                                                                   │
│     • Rollback: Undo the undo                                                                                               │
│     • Dry-run: ✓ supported                                                                                                  │
│                                                                                                                               │
│   Runbook Option 3 (Preventive):                                                                                            │
│     • Name: "Apply Resource Limits"                                                                                        │
│     • Prerequisites: deployment exists                                                                                      │
│     • Action: update deployment memory limit to 512Mi                                                                       │
│     • Blast radius: Low (just a limit)                                                                                      │
│     • Rollback: Remove or adjust limit                                                                                      │
│     • Dry-run: ✓ supported                                                                                                  │
│                                                                                                                               │
│   Store runbook options in incidents table → suggested_runbooks field                                                       │
│                                                                                                                               │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                       PHASE 7: FRONTEND DISPLAY & USER ACTION                                               │
├─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                                               │
│   Frontend polls job status and receives full incident context:                                                             │
│                                                                                                                               │
│   Incident Dashboard shows:                                                                                                 │
│                                                                                                                               │
│   ┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐   │
│   │ 🚨 INCIDENT: nginx-app Pods Crashing (nginx, default namespace)                                                   │   │
│   ├─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤   │
│   │                                                                                                                     │   │
│   │ Probable Cause:                                                                                                    │   │
│   │   🔴 OUT OF MEMORY (Confidence: 95%)                                                                              │   │
│   │   Pod eviction cascade due to memory pressure.                                                                     │   │
│   │   [Learn more] [Explain in detail]                                                                                │   │
│   │                                                                                                                     │   │
│   │ Recent Changes (Last 2 Hours):                                                                                     │   │
│   │   📦 Deployment rolled out: nginx-app v2.1 → v2.2                                                                 │   │
│   │   ⏰ 60 minutes ago by user@example.com                                                                             │   │
│   │   ⚠️  Correlation: HIGH (Memory increase started ~same time)                                                       │   │
│   │   [View diff]                                                                                                      │   │
│   │                                                                                                                     │   │
│   │ Timeline:                                                                                                           │   │
│   │   ├─ 60min ago: Deployment v2.2 rolled out                                                                        │   │
│   │   ├─ 45min ago: Memory usage increased from 200Mi → 450Mi                                                          │   │
│   │   ├─ 20min ago: First OOMKilled event                                                                              │   │
│   │   ├─ 10min ago: Node memory pressure                                                                               │   │
│   │   └─ Now:       Pod eviction cascade                                                                               │   │
│   │   [Expand timeline] [Export as PDF]                                                                                │   │
│   │                                                                                                                     │   │
│   │ Recommended Actions:                                                                                                │   │
│   │                                                                                                                     │   │
│   │   ┌─ Option 1: Rollback Deployment to v2.1 ──────────────────────────────────────────────────────────────────┐    │   │
│   │   │ Blast Radius: LOW | Confidence: HIGH | Rollback: Easy                                                    │    │   │
│   │   │ kubectl rollout undo deployment/nginx-app --namespace=default                                           │    │   │
│   │   │ [Run dry-run first] [Execute now] [View runbook details]                                                │    │   │
│   │   └──────────────────────────────────────────────────────────────────────────────────────────────────────────┘    │   │
│   │                                                                                                                     │   │
│   │   ┌─ Option 2: Scale Up Replicas (Temporary) ──────────────────────────────────────────────────────────────────┐   │   │
│   │   │ Blast Radius: MEDIUM | Confidence: MEDIUM | Rollback: Easy                                               │   │   │
│   │   │ kubectl scale deployment/nginx-app --replicas=5 --namespace=default                                      │   │   │
│   │   │ [Run dry-run first] [Execute now]                                                                        │   │   │
│   │   └──────────────────────────────────────────────────────────────────────────────────────────────────────────────┘   │   │
│   │                                                                                                                     │   │
│   │   ┌─ Option 3: Apply Resource Limits (Preventive) ──────────────────────────────────────────────────────────────┐   │   │
│   │   │ Blast Radius: LOW | Confidence: HIGH | Rollback: Easy                                                     │   │   │
│   │   │ Recommended limit: 512Mi (based on 80th percentile of observed usage)                                     │   │   │
│   │   │ [Run dry-run first] [Execute now]                                                                        │   │   │
│   │   └──────────────────────────────────────────────────────────────────────────────────────────────────────────────┘   │   │
│   │                                                                                                                     │   │
│   │ Explanation (Why This Happened):                                                                                   │   │
│   │   Memory leak in nginx-app v2.2. Check release notes for known issues.                                           │   │
│   │   [View pod logs] [Check image registry]                                                                         │   │
│   │                                                                                                                     │   │
│   └─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                                                               │
│   User clicks: "Execute now" → Rollback Deployment                                                                         │
│                                                                                                                               │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                          PHASE 8: DRY-RUN & APPROVAL                                                        │
├─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                                               │
│   Runbook Orchestrator Lambda (runbooks/{runbook_id}/dry-run)                                                               │
│                                                                                                                               │
│   1. Get current deployment state (via spoke role)                                                                          │
│   2. Simulate action on copy:                                                                                               │
│       kubectl --dry-run=client rollout undo deployment/nginx-app --namespace=default                                       │
│   3. Predict changes:                                                                                                       │
│       Rollback from v2.2 → v2.1                                                                                             │
│       Replicas: 3 → 3 (no change)                                                                                           │
│       Pods affected: 3 (all current pods will be replaced)                                                                  │
│   4. Return dry-run result to frontend                                                                                      │
│                                                                                                                               │
│   Frontend shows dry-run results:                                                                                           │
│   ┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐   │
│   │ Dry-Run Results:                                                                                                    │   │
│   │   ✓ Deployment exists                                                                                              │   │
│   │   ✓ Revision history available                                                                                     │   │
│   │   ✓ Will rollback to v2.1 (deployed 120 mins ago)                                                                  │   │
│   │   ✓ 3 pods will be replaced (estimated 30-45 seconds)                                                              │   │
│   │   ✓ Rollback command validated                                                                                     │   │
│   │                                                                                                                     │   │
│   │ [Confirm & Execute] [Cancel] [Show advanced options]                                                               │   │
│   └─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                                                               │
│   User clicks: "Confirm & Execute"                                                                                          │
│                                                                                                                               │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                               PHASE 9: EXECUTION                                                             │
├─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                                               │
│   Runbook Orchestrator Lambda (runbooks/{runbook_id}/execute)                                                               │
│                                                                                                                               │
│   1. Get execution approval (or auto-approve if LOW risk + permitted role)                                                   │
│   2. Create runbook job in DynamoDB                                                                                         │
│   3. Assume spoke role → execute actual kubectl command                                                                      │
│       kubectl rollout undo deployment/nginx-app --namespace=default                                                         │
│   4. Capture output + return code                                                                                           │
│   5. Emit OpenTelemetry span for action execution (X-Ray trace)                                                              │
│   6. Update DynamoDB runbook_jobs table:                                                                                    │
│       job_id, status=SUCCESS, executed_by, timestamp, command, output, rollback_steps                                      │
│   7. Update incidents table:                                                                                                │
│       remediation_applied=true, applied_at=<timestamp>, applied_runbook_id, outcome=pending                               │
│   8. Write audit log (who did what, when, why)                                                                              │
│                                                                                                                               │
│   Frontend polls and shows execution progress:                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐   │
│   │ Execution in Progress...                                                                                            │   │
│   │   ⏳ Executing: kubectl rollout undo deployment/nginx-app --namespace=default                                      │   │
│   │   ⏱ Estimated time: 30-45 seconds                                                                                  │   │
│   │   [Monitor] [Cancel] [View live logs]                                                                              │   │
│   │                                                                                                                     │   │
│   │ ✓ Command executed successfully!                                                                                   │   │
│   │ Rollback to v2.1 initiated. Pods will be replaced in ~45 seconds.                                                 │   │
│   │ [Monitor pod status] [View runbook log] [Next steps]                                                               │   │
│   └─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                                                               │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                         PHASE 10: MONITORING & SLO TRACKING                                                 │
├─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                                               │
│   After action execution, SLO Engine Lambda monitors recovery:                                                              │
│                                                                                                                               │
│   1. Check if incident resolved:                                                                                             │
│       • All pods Running? (YES)                                                                                             │
│       • All nodes healthy? (YES)                                                                                            │
│       • No OOMKilled events? (YES)                                                                                          │
│   2. Measure recovery time:                                                                                                  │
│       Issue reported: 12:00 PM                                                                                              │
│       Rollback executed: 12:05 PM                                                                                           │
│       Pods fully Running: 12:06 PM                                                                                          │
│       MTTR = 6 minutes ✓ (SLO target: 15 minutes)                                                                           │
│   3. Update SLO metrics:                                                                                                     │
│       Availability SLO: 99.5% → 99.6% (small improvement)                                                                  │
│       Error budget remaining: 2.1 hours                                                                                     │
│   4. Fire recovery alert (if configured):                                                                                    │
│       Slack: "✅ Incident resolved in 6 min via rollback (MTTR target met)"                                                │
│   5. Update incident status:                                                                                                │
│       status=RESOLVED, resolution_time=6min, remediation_outcome=SUCCESS                                                    │
│                                                                                                                               │
│   Frontend displays:                                                                                                         │
│   ┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐   │
│   │ ✅ INCIDENT RESOLVED                                                                                               │   │
│   │ Resolution Time: 6 minutes                                                                                         │   │
│   │ SLO Status: ✓ Met (target 15 min)                                                                                  │   │
│   │                                                                                                                     │   │
│   │ Remediation: Rollback deployment to v2.1                                                                          │   │
│   │ Executed: 12:05 PM by platform-admin                                                                              │   │
│   │                                                                                                                     │   │
│   │ Cluster Health: ✓ All Healthy                                                                                      │   │
│   │   • Pods: 3/3 Running                                                                                              │   │
│   │   • Nodes: 3/3 Ready                                                                                               │   │
│   │   • Memory: Normal                                                                                                  │   │
│   │                                                                                                                     │   │
│   │ [Generate post-mortem] [View full timeline] [View audit log]                                                       │   │
│   └─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                                                               │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                       PHASE 11: AUTO-SUMMARY & POST-MORTEM                                                  │
├─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                                               │
│   Summary Generator Lambda creates post-mortem narrative:                                                                    │
│                                                                                                                               │
│   ┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓    │
│   ┃ INCIDENT SUMMARY: nginx-app Pods Crashing - May 30, 2026                                                       ┃    │
│   ┣━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┫    │
│   ┃                                                                                                                  ┃    │
│   ┃ OVERVIEW                                                                                                        ┃    │
│   ┃ ────────                                                                                                        ┃    │
│   ┃ Service: nginx-app (production-default)                                                                        ┃    │
│   ┃ Duration: 12:00 PM - 12:06 PM (6 minutes)                                                                      ┃    │
│   ┃ Severity: P2 (High) - User-facing service degradation                                                           ┃    │
│   ┃ Root Cause: Memory leak in nginx-app v2.2 deployment                                                            ┃    │
│   ┃ Resolution: Rollback to v2.1                                                                                    ┃    │
│   ┃ Status: ✓ RESOLVED (SLO met)                                                                                    ┃    │
│   ┃                                                                                                                  ┃    │
│   ┃ TIMELINE                                                                                                        ┃    │
│   ┃ ────────                                                                                                        ┃    │
│   ┃ 12:00 PM - Detection: Prometheus alert triggered (pod restart rate > 10/min)                                   ┃    │
│   ┃ 12:00 PM - User initiates diagnostics via platform dashboard                                                   ┃    │
│   ┃ 12:01 PM - Root Cause Analysis: "Out of memory" (confidence 95%)                                               ┃    │
│   ┃ 12:01 PM - Change Correlation: nginx-app v2.2 deployment 60min prior                                           ┃    │
│   ┃ 12:02 PM - Runbook matched: "Rollback deployment" recommended                                                  ┃    │
│   ┃ 12:02 PM - Dry-run validated, user approved execution                                                          ┃    │
│   ┃ 12:05 PM - Rollback executed: deployment/nginx-app undo                                                        ┃    │
│   ┃ 12:06 PM - Recovery: All pods running, incident closed                                                         ┃    │
│   ┃                                                                                                                  ┃    │
│   ┃ ROOT CAUSE ANALYSIS                                                                                             ┃    │
│   ┃ ───────────────────                                                                                             ┃    │
│   ┃ nginx-app v2.2 contained a memory leak affecting connection handling.                                           ┃    │
│   ┃ Long-lived connections exhausted pod memory, triggering OOMKilled.                                              ┃    │
│   ┃                                                                                                                  ┃    │
│   ┃ Contributing factors:                                                                                            ┃    │
│   ┃ • No memory resource limits set on deployment (allowed unlimited growth)                                        ┃    │
│   ┃ • No pre-deployment memory regression testing                                                                   ┃    │
│   ┃ • No monitoring alert on pod memory usage (only restart rate)                                                   ┃    │
│   ┃                                                                                                                  ┃    │
│   ┃ REMEDIATION                                                                                                      ┃    │
│   ┃ ───────────                                                                                                     ┃    │
│   ┃ Immediate: Rollback to v2.1 (executed 12:05 PM by platform-admin)                                              ┃    │
│   ┃ Short-term: Apply memory resource limit 512Mi to deployment                                                     ┃    │
│   ┃ Long-term: Add memory regression testing to CI/CD; set memory alerts per pod                                    ┃    │
│   ┃                                                                                                                  ┃    │
│   ┃ PREVENTION                                                                                                       ┃    │
│   ┃ ──────────                                                                                                      ┃    │
│   ┃ • Update deployment policy: all workloads must have resource requests/limits                                    ┃    │
│   ┃ • Add SLO: pod memory alert when usage > 80% of limit                                                           ┃    │
│   ┃ • Enhance CI/CD: automatic memory profiling before merge (golang pprof, etc)                                    ┃    │
│   ┃ • Tune compliance checks to flag missing resource limits                                                         ┃    │
│   ┃                                                                                                                  ┃    │
│   ┃ IMPACT METRICS                                                                                                   ┃    │
│   ┃ ──────────────                                                                                                  ┃    │
│   ┃ MTTR: 6 minutes (SLO target: 15 min) ✓ MET                                                                      ┃    │
│   ┃ Availability impact: ~0.01% (6 min / 12 hours = 0.008%)                                                         ┃    │
│   ┃ Users affected: ~500 (estimated based on traffic)                                                               ┃    │
│   ┃ Service: nginx-app (critical path: API gateway → nginx-app → database)                                          ┃    │
│   ┃                                                                                                                  ┃    │
│   ┣━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┫    │
│   ┃ Generated: 2026-05-30 12:07 PM | Audit log ID: incident-12345 | Report ID: postmortem-12345                     ┃    │
│   ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛    │
│                                                                                                                               │
│   Frontend options:                                                                                                          │
│   [Export as Markdown] [Export as PDF] [Share via Slack] [Create Jira Ticket] [Print]                                      │
│                                                                                                                               │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Component Interaction Matrix

| Component | Communicates With | Protocol | Purpose |
|-----------|-------------------|----------|---------|
| Frontend | API Gateway | HTTPS + JWT | UI requests, incident queries |
| API Gateway | Lambda Functions | Direct invoke | Route requests to handlers |
| Lambda Functions | EKS API | kubectl/Python client | Cluster diagnostics |
| Lambda Functions | DynamoDB | Boto3 | Store incidents, timelines, jobs |
| Lambda Functions | CloudWatch | Logs API | Structured logging |
| Lambda Functions | X-Ray | OpenTelemetry SDK | Distributed tracing |
| Lambda Functions | IAM/STS | Boto3 | Assume spoke roles |
| CloudWatch | Dashboard/Alarms | CloudWatch API | Metrics and alerting |
| X-Ray | Service Map | X-Ray API | Dependency visualization |
| External Integrations | Lambda/API | Webhooks + REST | Slack, Jira, GitHub |

---

## Key Features & Capabilities Summary

### Core Diagnostics
✅ **Multi-source metrics** (metrics-server, kubelet, CloudWatch)
✅ **Cross-account access** via spoke role assumption
✅ **Full RBAC validation** (service accounts, role bindings)
✅ **Event timeline aggregation** (kubernetes events + audit logs)
✅ **Resource state snapshot** (pod, node, storage, network)

### Intelligence & Analysis
✅ **Root Cause Engine** (hypothesis scoring with confidence)
✅ **Incident Timeline** (chronological narrative with correlation)
✅ **Change Correlation** (links deployments, config changes to incidents)
✅ **Runbook Matching** (safe action suggestions with risk classification)
✅ **Auto-Summary** (post-mortem generation)

### Governance & Safety
✅ **Fine-grained RBAC** (permissions tied to roles and actions)
✅ **Dry-run validation** (preview changes before execution)
✅ **Approval gates** (privileged actions require review)
✅ **Audit logging** (all changes tracked with who/what/when/why)
✅ **Compliance automation** (policy-as-code checks)

### Observability & Performance Tuning
✅ **OpenTelemetry instrumentation** (all Lambda paths traced)
✅ **X-Ray service mapping** (dependencies and latency)
✅ **CloudWatch dashboards** (custom metrics and logs)
✅ **SLO tracking** (burn rate, error budget, incident correlation)
✅ **APM-ready** (compatible with DataDog, Dynatrace, New Relic)

### Enterprise Integrations
✅ **Slack/Teams notifications** (incident alerts, action confirmations)
✅ **Jira/ServiceNow sync** (bi-directional ticket creation and updates)
✅ **GitHub/GitLab webhooks** (ingest deployment and code events)
✅ **Custom webhook brokers** (extensible integration pattern)

---

## Deployment & Operational Model

```
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ DEPLOYMENT ARCHITECTURE                                                                                                     │
├─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                                               │
│ Development:                                                                                                                │
│   • Code repo: GitHub (source of truth)                                                                                    │
│   • CI/CD: GitHub Actions → push to dev environment                                                                        │
│   • Infrastructure: Terraform (IaC for Lambda, API Gateway, DynamoDB, Cognito)                                             │
│   • Testing: pytest (unit) + integration tests (real cluster)                                                               │
│                                                                                                                               │
│ Staging:                                                                                                                    │
│   • Separate AWS account or sub-account for safety                                                                         │
│   • Same infrastructure as prod (Terraform apply to staging)                                                               │
│   • Reduced traffic/retention policies for cost control                                                                    │
│   • Full integration tests with staging EKS cluster                                                                         │
│                                                                                                                               │
│ Production:                                                                                                                 │
│   • AWS Account: 307946672793 (us-east-2)                                                                                  │
│   • Lambda functions: all cold-start optimized                                                                             │
│   • DynamoDB: on-demand billing (scale with usage)                                                                         │
│   • Cognito: production user pool with multi-factor auth                                                                   │
│   • CloudFront: cached static frontend with origin shield                                                                  │
│   • Monitoring: CloudWatch Insights, X-Ray, custom dashboards                                                              │
│                                                                                                                               │
│ Disaster Recovery:                                                                                                          │
│   • DynamoDB: point-in-time recovery enabled (35-day retention)                                                            │
│   • Code: version-controlled in GitHub with branch protection                                                              │
│   • Infrastructure: Terraform state backed up to S3                                                                         │
│   • Runbook execution: audit trail in DynamoDB and CloudWatch                                                              │
│                                                                                                                               │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Performance & Cost Considerations

### Typical Request Flow Latency
- User dashboard load: 200-500ms (CloudFront cached)
- API call to Lambda: 50-100ms (API Gateway overhead)
- Diagnostics collection (small cluster): 1-5 seconds
- Root cause analysis: 500-2000ms
- Total dashboard update: 2-8 seconds

### Cost Breakdown (Monthly Estimate)
| Component | Usage | Cost |
|-----------|-------|------|
| Lambda invocations | 100,000 | ~$20 |
| API Gateway | 100,000 requests | ~$35 |
| DynamoDB | 1 GB stored, 10 RCU/5 WCU avg | ~$15 |
| CloudWatch Logs | 10 GB ingested | ~$50 |
| X-Ray traces | 10,000 traces/month | ~$25 |
| CloudFront | 1 TB out | ~$85 |
| **Total** | | **~$230/month** |

---

This specification provides a complete reference for the platform architecture, use cases, data flows, and operational model. All components are designed for scalability, safety, and cost efficiency.
