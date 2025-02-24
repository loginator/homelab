# Homelab

This repository contains definitions and documentation for my homelab (Currently only for my kubernetes cluster based on Talos)

# Talos Cluster

## Generating the image
You can just download the image from https://factory.talos.dev/image/ce4c980550dd2ab1b17bbf2b08801c7eb59418eafe8f279833297925d67c7515/v1.9.2/metal-amd64.iso

Upload it to proxmox to proceed.

## Setup nodes
### Creating the templates
Create a new node with the following configuration (Only non default values are mentioned):
- General:
  - Start at boot: **true**
- Disks:
  - Disk size: 64GiB
  - SSD Emulation: **true**
  - Discard: **true**
- CPU:
  - CPU Total Cores: 4
- Memory:
  - 4096 MiB
Right click the node and click convert to template

Create a new node with the following configuration (Only non default values are mentioned):
- General:
  - Start at boot: **true**
- Disks:
  - scsi0:
    - Disk size: 64GiB
    - SSD Emulation: **true**
    - Discard: **true**
  - scsi1 (add new disk):
    - Disk size: 128GiB
    - SSD Emulation: **true**
    - Discard: **true**
- CPU:
  - CPU Total Cores: 8
- Memory:
  - 16384 MiB 
Right click the node and click convert to template

## Creating the nodes
From the created template create 3 Master nodes and three worker nodes. Make sure to select **Full Clone** as the mode.
Now start the nodes at least once so they get an IP assigned by your DHCP Server. You should be able to see their ips in the dashboard (Console in proxmox).

Go ahead and asign them static ips in your DHCP servers config.
To restart all nodes you can enter CTRL+ALT+Delete to the system. Now your ips should be set.

### Generate configs
To generate configs run the following commands:
```bash
talosctl gen secrets -o secrets.yaml
talosctl gen config --with-secrets .\secrets\talos.secrets.yaml --config-patch-control-plane @talos/controlplane.patches.yaml taloscluster https://10.10.20.120:6443
```
> **Note:** Make sure to backup your secrets.yaml file somewhere save!

### Apply the config to the cluster
```bash
talosctl apply-config --insecure --nodes 10.10.20.121 --file ./controlplane.yaml
talosctl apply-config --insecure --nodes 10.10.20.122 --file ./controlplane.yaml
talosctl apply-config --insecure --nodes 10.10.20.123 --file ./controlplane.yaml
talosctl apply-config --insecure --nodes 10.10.20.124 --file ./worker.yaml
talosctl apply-config --insecure --nodes 10.10.20.125 --file ./worker.yaml
talosctl apply-config --insecure --nodes 10.10.20.126 --file ./worker.yaml
```
All machines should reboot and install their kubelet container.

### Bootstrap etcd
```bash
talosctl bootstrap -n 10.10.20.121
```

## Get kubeconfig
```bash
talosctl kubeconfig . -n 10.10.20.121
```

## Wipe disks using the talosctl
```bash
talosctl wipe disk sdb -n 10.10.20.124
```

# FluxCD
FluxCD is used to apply all of the configs

## Bootstrap the flux
```bash
export GITHUB_TOKEN=<token>
flux check --pre
flux bootstrap github --token-auth --owner=loginator --repository=homelab --branch=main --path=./clusters/production --personal
```

# Sealed Secrets

This repository uses Sealed Secrets to securely create and manage Kubernetes secrets, allowing you to safely store them in version control systems like GitHub.

## Install kubeseal

### Linux

To install `kubeseal`, follow these steps:

1. Set the `KUBESEAL_VERSION` environment variable to the desired version, for example:

    ```bash
    KUBESEAL_VERSION='0.28.0'
    ```

2. Download the `kubeseal` binary:

    ```bash
    curl -OL "https://github.com/bitnami-labs/sealed-secrets/releases/download/v${KUBESEAL_VERSION:?}/kubeseal-${KUBESEAL_VERSION:?}-linux-amd64.tar.gz"
    ```

3. Extract the binary from the tarball:

    ```bash
    tar -xvzf kubeseal-${KUBESEAL_VERSION:?}-linux-amd64.tar.gz kubeseal
    ```

4. Install the binary to `/usr/local/bin`:

    ```bash
    sudo install -m 755 kubeseal /usr/local/bin/kubeseal
    ```

### Windows

To install `kubeseal` on Windows using PowerShell, follow these steps:

1. Open PowerShell as an administrator.

2. Set the `KUBESEAL_VERSION` environment variable to the desired version:

    ```powershell
    $env:KUBESEAL_VERSION = "0.28.0"
    ```

3. Define the download URL, output file paths, and installation path:

    ```powershell
    $downloadUrl = "https://github.com/bitnami-labs/sealed-secrets/releases/download/v$env:KUBESEAL_VERSION/kubeseal-$env:KUBESEAL_VERSION-windows-amd64.tar.gz"
    $outputFile = "kubeseal-$env:KUBESEAL_VERSION-windows-amd64.tar.gz"
    $extractFolder = "kubeseal_extracted"
    $installPath = "C:\Tools"
    ```

4. Download the `kubeseal` binary:

    ```powershell
    Invoke-WebRequest -Uri $downloadUrl -OutFile $outputFile
    ```

5. Create a folder for extraction:

    ```powershell
    New-Item -ItemType Directory -Path $extractFolder
    ```

6. Extract the `.tar.gz` file:

    ```powershell
    tar -xvf $outputFile -C $extractFolder
    ```

7. Move the binary to the installation path:

    ```powershell
    Move-Item -Path "$extractFolder\kubeseal.exe" -Destination "$installPath\kubeseal.exe"
    ```

8. Add the installation path to your `PATH` environment variable if it's not already included:

    ```powershell
    [System.Environment]::SetEnvironmentVariable("Path", $env:Path + ";$installPath", [System.EnvironmentVariableTarget]::Machine)
    ```

9. Verify the installation by checking the `kubeseal` version:

    ```powershell
    kubeseal --version
    ```

10. Clean up the extracted files and the archive:

    ```powershell
    Remove-Item -Path $outputFile
    Remove-Item -Path $extractFolder -Recurse
    ```

## Get the Public Sealed-Secrets Key

The public key is used to generate Sealed Secrets from a standard Kubernetes secret. To fetch the public key, run the following command:

```bash
kubeseal --fetch-cert --controller-name=sealed-secrets-sealed-secrets --controller-namespace=sealed-secrets > pub-sealed-secrets.pem
```

## Encrypt a Secret

Follow these steps to encrypt a Kubernetes secret:

### Linux

1. Create a standard Kubernetes secret:

    ```bash
    kubectl -n default create secret generic basic-auth --from-literal=user=admin --from-literal=password=change-me --dry-run=client -o yaml > basic-auth.yaml
    ```

2. Encrypt the secret using the public key and then delete the original secret file to avoid accidentally pushing it to the repository:

    ```bash
    kubeseal --format=yaml --controller-name=sealed-secrets-sealed-secrets --controller-namespace=sealed-secrets < basic-auth.yaml > basic-auth-sealed.yaml
    rm basic-auth.yaml
    ```

The resulting `basic-auth-sealed.yaml` file contains the encrypted secret, which can be safely committed to your version control system.

### Windows

1. Create a standard Kubernetes secret:

    ```powershell
    kubectl -n default create secret generic basic-auth --from-literal=user=admin --from-literal=password=change-me --dry-run=client -o yaml > basic-auth.yaml
    ```

2. Encrypt the secret using the public key and then delete the original secret file to avoid accidentally pushing it to the repository:

    ```powershell
    Get-Content basic-auth.yaml | kubeseal --format=yaml --controller-name=sealed-secrets-sealed-secrets --controller-namespace=sealed-secrets | Set-Content basic-auth-sealed.yaml
    Remove-Item basic-auth.yaml
    ```

The resulting `basic-auth-sealed.yaml` file contains the encrypted secret, which can be safely committed to your version control system.

## Backup Private Keys

It is crucial to back up your private keys to ensure you can restore your Sealed Secrets in case your cluster is destroyed. Follow these steps to back up the private keys:

1. Retrieve the private keys from the `sealed-secrets` namespace:

    ```bash
    kubectl get secret -n sealed-secrets -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml > main.key
    ```

2. Store the `main.key` file in a secure location, such as a password-protected storage service or an encrypted backup.

By backing up your private keys, you can recreate your Sealed Secrets in a new cluster if needed.

## Restore Your Private Keys

In the event of a disaster recovery, you can restore your Sealed Secrets private keys by following these steps:

1. Apply the backed-up private key secret to your cluster:

    ```bash
    kubectl apply -f main.key
    ```

2. Restart the Sealed Secrets controller to ensure it picks up the restored keys:

    ```bash
    kubectl delete pod -n sealed-secrets -l app.kubernetes.io/name=sealed-secrets
    ```

By restoring your private keys, you can decrypt your Sealed Secrets in the new cluster.

## Additional Information

Sealed Secrets is a Kubernetes controller and tool that allows you to encrypt secrets and store them safely in a Git repository. The Sealed Secrets controller ensures that only the controller can decrypt the secrets, providing an additional layer of security.

For more information, visit the [Sealed Secrets GitHub repository](https://github.com/bitnami-labs/sealed-secrets).

# Authentik

## Initial setup
https://auth.local.sandrolab.net/if/flow/initial-setup/

## Create a proxy application

1. Visit https://auth.local.sandrolab.net/if/admin/#/core/providers and enter the following values:
    - Name: traefik-dashboard
    - Authorization flow: default-provider-authorization-explicit-consent (Authorize Application)
    - Foward auth (single application)
    - External host
    - https://traefik-dashboard.local.sandrolab.net
2. Visit https://auth.local.sandrolab.net/if/admin/#/core/applications and create a new application with the following values:
    - Name: traefik-dashboard
    - Slug: traefik-dashboard
    - Provider: traefik-dashboard
3. Visit https://auth.local.sandrolab.net/if/admin/#/outpost/outposts and add traefik-dashboard to the selected applications