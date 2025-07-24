## Generate a gpg key

```
gpg --batch --full-generate-key <<EOF
%no-protection
Key-Type: 1
Key-Length: 4096
Subkey-Type: 1
Subkey-Length: 4096
Expire-Date: 0
Name-Comment: "argocd secret management using ksops"
Name-Real: "argocd-ksops"
EOF
```

## Retrieve the key

```
gpg --list-secret-keys "argocd-ksops"
```

## Store the key fingerprint as an environment variable

```
export GPG_ID=DE35A15185CFDB8541D8A86DE1367C66E558CF5B
```

## Export the key pair to key.asc

```
gpg --export-secret-keys --armor "${GPG_ID}" > key.asc
```

## Note: key.asc should not be stored in the repo.

## Export the public key to encrypt secrets locally.

```
gpg --export --armor "${GPG_ID}" > key.pub.asc
```

## Export the key pair and create a secret for ArgoCD to read them

```
gpg --export-secret-keys --armor "${GPG_ID}" |
kubectl create secret generic sops-gpg \
--namespace=argocd \
--from-file=sops.asc=/dev/stdin
```

## Another way to create a secret for ArgoCD to be used for decrypting secrets.

```
kubectl create secret generic sops-gpg \
--namespace=argocd \
--from-file=sops.asc=./key.asc
```

## Check the running ArgoCD version and replace the image version in repo-server-patch.yaml

```
kubectl get deployments -n argocd argocd-server -oyaml | yq '.spec.template.spec.containers[0].image'
```

## Apply the cm patch

```
kubectl apply -f cm-patch.yaml
```

## Apply the repo server patch

```
kubectl patch deployment -n argocd argocd-repo-server --patch-file ./repo-server-patch.yaml
```

## Restart components

```
kubectl rollout restart deployment -n argocd argocd-repo-server argocd-server

kubectl rollout restart sts -n argocd argocd-application-controller
```

## Encrypt secret.yaml with sops

```
sops --encrypt -p "${GPG_ID}" secret.yaml > app/secret.enc.yaml
```

## Check if sops can decrypt the secret

```
sops --decrypt -p "${GPG_ID}" app/secret.enc.yaml
```

## Test the ksops integration by pushing this repo to a Git forge and creating an Application using app as your path





