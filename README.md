# devops-kube-state

Deklarativno stanje klastera (GitOps preko ArgoCD). App-of-apps root: `clusters/local/root.yaml`.
Child aplikacije u `clusters/local/apps/` (sync-wave: 0 infra, 1 operator, 2 shophub).