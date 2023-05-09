## Create configmap containing CAs / self-signed-certs

> NOTE: the INIT_SCRIPT later at delegate installation time will only try to trust .crt files; make sure this is accounted for in any file names below

```bash
# !!! change delegate_ns if needed
delegate_ns="harness-delegate-ng"

kubectl create configmap -n "${delegate_ns}" ca-certs \
  --from-file=harness-ca-root.crt \
  --from-file=harness-ca-inter-a.crt \
  --from-file=harness-ca-inter-b.crt \
  --dry-run=client -o yaml \
  > configmap_ca-certs.yaml

kubectl apply -f configmap_ca-certs.yaml
```

## Install / upgrade delegate with INIT_SCRIPT and extra env vars to trust certificates

```bash
# !!! change delegate_name
delegate_name="DELEGATE_NAME_GOES_HERE"

cat <<EOF > post-render.sh
#!/bin/bash

cat <&0 > all.yaml

kustomize && rm all.yaml
EOF

cat <<EOF > kustomization.yaml
resources:
  - all.yaml
patches:
  - path: patch-delegate-configmap.yaml
    target:
      kind: ConfigMap
      name: "${delegate_name}"
  - path: patch-delegate-deployment.yaml
    target:
      kind: Deployment
      name: "${delegate_name}"
EOF

cat <<EOF > patch-delegate-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: _
data:
  INIT_SCRIPT: |
    jre_path="/opt/java/openjdk"
    mkdir -p /tmp/harness/ca-certs/
    
    for f in /tmp/ca-certs/*.crt ; do
      no_prefix="\${f#/tmp/ca-certs/}"
      id="\${no_prefix%.crt}"
      echo "adding cert \${id} to trust store"
    
      # create bundle of all CAs
      echo "\${id}" >> /tmp/harness/ca-certs/cacerts.pem
      cat "\${f}" >> /tmp/harness/ca-certs/cacerts.pem
      echo "" >> /tmp/harness/ca-certs/cacerts.pem
    
      # copy target cert to UBI CA certs location
      cp "\${f}" /etc/pki/ca-trust/source/anchors
    
      # add target cert to Java trust store
      "\${jre_path}/bin/keytool" -import -trustcacerts -keystore "\${jre_path}/lib/security/cacerts" -storepass changeit -alias "\${id}" -file "\${f}" -noprompt
    done
    update-ca-trust
  ADDITIONAL_CERTS_PATH: "/tmp/harness/ca-certs/cacerts.pem"
  CI_MOUNT_VOLUMES: "/etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem:/etc/ssl/certs/ca-bundle.crt,/tmp/harness/ca-certs/cacerts.pem:/kaniko/ssl/certs/additional-ca-cert-bundle.crt"
EOF

cat <<EOF > patch-delegate-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: _
spec:
  template:
    spec:
      containers:
        - name: delegate
          volumeMounts:
            - name: ca-certs
              mountPath: /tmp/ca-certs
      volumes:
        - name: ca-certs
          configMap:
            name: ca-certs
EOF

chmod +x post-render.sh

helm repo add harness-delegate https://app.harness.io/storage/harness-download/delegate-helm-chart/
helm repo update

helm upgrade --install \
  "${delegate_name}" harness-delegate/harness-delegate-ng \
  -n harness-delegate-ng --create-namespace \
  -f values.yaml \
  --post-renderer ./post-render.sh
```
