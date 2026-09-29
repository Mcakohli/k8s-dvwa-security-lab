# Cloud-Native Security Lab: DVWA on Kubernetes & Runtime Threat Analysis

A hands-on implementation demonstrating application vulnerability analysis, container security implications, and runtime zero-trust defense on a local Kubernetes cluster.

---

## 🛠️ Architecture & Deployment

The application is deployed on a local single-node Kubernetes cluster (Minikube / Docker driver).

### Prerequisites
- Docker Engine
- Minikube
- `kubectl`

### Cluster Verification State
The deployment runs within an isolated `dvwa-lab` namespace exposed via a `NodePort` service on port `30080`.

```bash
kubectl get pods,svc -n dvwa-lab
```

<img src="./Screen%20shots/Cluster%20verification%20terminal.png" alt="Cluster Verification Terminal" width="850" />

---

## 🎯 Explored Attack Surfaces

### 1. Command Injection (Remote Code Execution & Pod Token Theft)
- **Endpoint:** `/vulnerabilities/exec/`
- **Vulnerability:** Unsanitized user input passed directly to the underlying OS shell.
- **Payload:**
  ```bash
  127.0.0.1; whoami; id; uname -a; cat /var/run/secrets/kubernetes.io/serviceaccount/token
  ```
- **Exploitation:** Executed commands under the `www-data` service account and extracted the mounted Kubernetes Service Account JWT token from `/var/run/secrets/kubernetes.io/serviceaccount/token`.
- **K8s Blast Radius:** Enables unauthorized API interrogation and privilege escalation against the cluster control plane (`kube-apiserver`).

<img src="./Screen%20shots/Command%20injection.png" alt="Command Injection Exploit" width="850" />

---

### 2. SQL Injection (Union-Based Credential Extraction)
- **Endpoint:** `/vulnerabilities/sqli/`
- **Vulnerability:** Unescaped parameters concatenated into raw database queries.
- **Payload:**
  ```sql
  ' UNION SELECT user, password FROM users #
  ```
- **Exploitation:** Extracted backend database records containing usernames and MD5 password hashes (`admin`, `Gordon`, `1337`, `pablo`).
- **Impact:** Complete backend credential compromise enabling offline hash cracking.

<img src="./Screen%20shots/SQL%20injection.png" alt="SQL Injection Exploit" width="850" />

---

### 3. Cross-Site Scripting (Reflected XSS)
- **Endpoint:** `/vulnerabilities/xss_r/`
- **Vulnerability:** Direct unescaped reflection of user input into the browser DOM.
- **Payload:**
  ```html
  <script>alert('XSS-Executed: ' + document.cookie)</script>
  ```
- **Exploitation:** Executed client-side JavaScript executing in the active session context, triggering an alert dialog exposing `PHPSESSID`.
- **Impact:** Session hijacking, cookie theft, and administrative CSRF.

<img src="./Screen%20shots/Reflected%20XSS.png" alt="Reflected XSS Exploit" width="850" />

---

## 🛡️ Cloud-Native Defense & Zero-Trust Mitigation

While web application firewalls (WAF) and code fixes address application bugs, cloud-native workloads require **runtime zero-trust enforcement** to prevent container breakout and lateral movement:

1. **Inline Process Blocking (KubeArmor / LSMs):**  
   Apply a `KubeArmorPolicy` enforcing execution controls. Even if command injection succeeds at the PHP layer, KubeArmor intercepts process execution at the Linux kernel level (via BPF-LSM / AppArmor) and blocks `/bin/sh`, `/bin/bash`, or `cat` from spawning.
2. **Credential Hardening:**  
   Enforce `automountServiceAccountToken: false` on pod specs that do not need to interact with the Kubernetes API, denying attackers access to default cluster secrets.
3. **Container Immutability:**  
   Set `securityContext.readOnlyRootFilesystem: true` and drop unnecessary Linux capabilities (`CAP_NET_RAW`, `CAP_SYS_ADMIN`) to prevent persistence and privilege escalation.

---

## 👤 Author
- **Rahul Kohli**
- [GitHub Profile](https://github.com/McaKohli)
