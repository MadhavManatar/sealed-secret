# Sealed Secrets Implementation Guide
## Overview
Sealed Secrets is a tool designed for encrypting Kubernetes Secrets, allowing secure passage of sensitive information to containers as environment variables or regular files when mounting secret volumes. This implementation involves two main components:

    Cluster-Side Controller / Operator


### Kubeseal CLI Tool

#### Step 1: Install Sealed Secrets Controller
To use Sealed Secrets, you must first install the controller on your Kubernetes cluster using the provided controller.yml file:


    kubectl apply -f 1.controller-v0-21-0.yml

This command creates a Sealed Secrets controller and a secret key for decryption within the kube-system namespace.

#### Step 2: Install Kubeseal on Your Local Machine

Install the Kubeseal CLI tool on your local machine using the following commands:

    KUBESEAL_VERSION=$(curl -s https://api.github.com/repos/bitnami-labs/sealed-secrets/tags | jq -r '.[0].name' | cut -c 2-)

    wget "https://github.com/bitnami-labs/sealed-secrets/releases/download/v${KUBESEAL_VERSION}/kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz"

    tar -xvzf kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz kubeseal

    sudo install -m 755 kubeseal /usr/local/bin/kubeseal

#### Step 3: Encrypt the Secrets
Create a Kubernetes secrets.yml file containing necessary environment variables (base64 encoded values).

#### Generate the SealedSecrets file using Kubeseal:

    kubeseal --format=yaml < secrets.yml > ../sealedsecrets.yml

#### Apply the sealed secrets to your cluster:

    kubectl apply -f sealedsecrets.yml

Check the status of unsealed secrets:

    kubectl describe sealedsecrets -n namespace

Examples in Deployments
Integrate Sealed Secrets into your deployment files with examples like:

    secretBackend:
      externalSecret:
          name: "my-secret"

    ---

    - name: username
        valueFrom:
          secretKeyRef:
            name: my-secret
            key: username

### Key Rotation
Sealed Secrets supports automatic key rotation every 30 days. Modify the rotation period using:

    --key-renew-period=240h

#### To manually rotate the key:

Generate a new key pair:


    openssl req -x509 -days 365 -nodes -newkey rsa:4096 -keyout "secret-private-key" -out "secret-public-key" -subj "/CN=sealed-secret/O=sealed-secret"

### Create a secret to store both public and private keys:

    kubectl -n kube-system create secret tls "sealed-secret" --cert="secret-public-key" --key="secret-private-key"

#### Activate the new key:

    kubectl -n kube-system label secret "sealed-secret" sealedsecrets.bitnami.com/sealed-secrets-key=active

#### Delete and reapply the SealedSecret controller:

    kubectl delete deployment sealed-secrets-controller -n kube-system
    kubectl apply -f controller-v0-21-0.yml

##### Re-encrypt existing SealedSecrets:

    kubeseal --format=yaml --re-encrypt -o yaml  < secrets.yml > ../sealedsecrets.yml
    Backup Key

#### For backup purposes, create a backup file of all encrypted keys:

    kubectl -n kube-system get secret -l sealedsecrets.bitnami.com/sealed-secrets-key=active -o yaml > allsealkeys.yml

#### To use the keys on a different cluster, apply the backup and re-encrypt:

    kubeseal rotate --all

This refined guide provides a step-by-step approach for implementing Sealed Secrets in Kubernetes, including key rotation and backup procedures. If you have any questions or need further clarification, feel free to ask.
