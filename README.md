# Maximus Applications
maximus-apps

```
kubectl apply -f - <<EOF
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: GitRepository
metadata:
  name: maximus-catalog-applications
  namespace: kommander
  labels:
    kommander.d2iq.io/gitapps-gitrepository-type: catalog
    kommander.d2iq.io/gitrepository-type: catalog
    kommander.d2iq.io/workspace-default-catalog-repository: "true"
    kommander.d2iq.io/project-default-catalog-repository: "true"
spec:
  interval: 1m0s
  ref:
    branch: main
  timeout: 20s
  url: https://github.com/mshayda/maximus-apps
EOF
```