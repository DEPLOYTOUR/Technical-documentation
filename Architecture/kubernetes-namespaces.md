# Kubernetes Namespaces Distribution

To ensure security, logical segregation, and efficient resource management, the Deploytour dataspace cluster is structured into distinct Kubernetes namespaces. This separation isolates administrative and central components from participant environments.

```mermaid
graph TD
    subgraph Cluster ["Deploytour Kubernetes Cluster"]
        argocd["Namespace: argocd<br>(GitOps CD)"]
        operator["Namespace: operator<br>(Governance & Central Services)"]
        tools["Namespace: tools<br>(Keycloak & Telemetry/Monitoring)"]
        participant["Namespace: participant-edc-&lt;name&gt;<br>(Sovereign Connector & Wallet)"]
    end

    Vault[(External HashiCorp Vault)]

    %% Connections to Vault
    operator -.->|Authenticate & Access| Vault
    participant -.->|Authenticate & Access| Vault
```

---

## Namespaces Breakdown

### 1. `argocd`
This namespace is dedicated entirely to the deployment and operation of **ArgoCD**. It manages the Continuous Delivery (CD) pipeline of the dataspace, implementing GitOps principles to synchronize the cluster's state with the git repository configuration.
* **Key Components**: ArgoCD Application Controller, Repo Server, and API Server.

### 2. `operator`
Hosts the central governance and administrative applications of the dataspace. These are operated by the central authority to orchestrate onboarding, compliance, and catalog federation.
* **Hosted Components**:
  * **Onboarding Portal**: Web GUI for user onboarding and administrator oversight.
  * **Onboarding Backend**: Business logic orchestration.
  * **Federated Catalog**: Aggregates asset metadata across the dataspace.
  * **Authority Identity Hub**: Manages Central Decentralized Identifiers (DIDs) and issues Verifiable Credentials (VCs).

### 3. `tools`
Contains shared backing utilities and monitoring systems. These services support the cluster's security and observability.
* **Hosted Components**:
  * **Keycloak**: Central Identity Provider (IdP) for web portals and API clients.
  * **External HashiCorp Vault**: Used for secure storage of secrets and cryptographic keys (hosted externally to the Kubernetes cluster to optimize resources and enhance security boundaries).
  * **Audit & Telemetry**: OpenTelemetry collector and ClickHouse database for high-performance log and audit trail analysis.
  * **Monitoring Stack**: Prometheus (metrics ingestion), Grafana (visualization), Loki (log aggregation), and Tempo (distributed tracing).

### 4. `participant-edc-<name>`
A dynamically provisioned namespace created **for each participant** who joins the dataspace. This guarantees absolute runtime isolation, preventing participants from accessing each other's pods, secrets, or configurations.
* **Naming Convention**: `participant-edc-<participant-name>` (e.g., `participant-edc-participant1`).
* **Hosted Components**:
  * **Participant Identity Hub**: Wallet managing the participant's Verifiable Credentials (VCs).
  * **GUI Connector**: Graphical interface for managing connector assets, policies, and contracts.
  * **EDC Connector**: Composed of the Control Plane (negotiations and policies) and the Data Plane (secure data channel).
