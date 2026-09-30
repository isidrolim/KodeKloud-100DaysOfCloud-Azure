# 100 Days of Cloud Azure - Day 023: Automating User Data Configuration Using the CLI

## Scenario

The Nautilus DevOps team needs to deploy an Azure Virtual Machine that will serve as an Nginx web server.

Instead of manually configuring the server after deployment, the VM must be provisioned through the **Azure CLI** and automatically install and start Nginx using a startup script during its initial boot.

The web server must also be accessible from the internet over HTTP port `80`.

## Requirement

Create the Azure VM with the following configuration:

- **VM Name:** `datacenter-vm`
- **Region:** `East US`
- **Image:** Any available Ubuntu image
- **Deployment Method:** Azure CLI
- **Web Server:** Nginx
- **Configuration:** Custom Script / User Data
- **HTTP Port:** `80`
- **HTTP Access:** Internet
- Use the default resource group or an existing resource group if needed.

The startup script must:

- Install Nginx.
- Start the Nginx service.
- Leave Nginx running after provisioning.

## Implementation

### 1. Identify the Existing Resource Group

Before creating resources, identify the lab resource group:

```bash
az group list --output table
```

Store the required resource group in a variable:

```bash
RG="<resource-group-name>"
```

Confirm it:

```bash
echo "$RG"
```

---

## 2. Create the Startup Script

Create a local cloud-init/startup script:

```bash
cat > /tmp/nginx-init.sh <<'EOF'
#!/bin/bash
apt-get update
apt-get install -y nginx
systemctl enable nginx
systemctl start nginx
EOF
```

Verify the script before using it:

```bash
cat /tmp/nginx-init.sh
```

The provisioning path will be:

```text
Azure VM boots
      ↓
Startup script executes
      ↓
apt-get update
      ↓
Install Nginx
      ↓
Enable Nginx
      ↓
Start Nginx
```

---

## 3. Create the Azure VM

Create the VM in `East US` using an available Ubuntu image:

```bash
az vm create \
  --resource-group "$RG" \
  --name datacenter-vm \
  --location eastus \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --generate-ssh-keys \
  --custom-data /tmp/nginx-init.sh
```

> The challenge allows any available Ubuntu image. The exact image or VM size can be adjusted if the lab subscription restricts a particular SKU.

Wait for the VM deployment to complete successfully.

---

## 4. Allow HTTP Traffic

Create an inbound NSG rule allowing internet traffic on TCP port `80`:

```bash
az vm open-port \
  --resource-group "$RG" \
  --name datacenter-vm \
  --port 80 \
  --priority 1010
```

This establishes the required network path:

```text
Internet
   │
   │ TCP/80
   ▼
Public IP
   │
   ▼
Network Security Group
   │
   │ Allow HTTP/80
   ▼
VM Network Interface
   │
   ▼
datacenter-vm
   │
   ▼
Nginx
```

## Verification

### Verify the VM State

Check the VM:

```bash
az vm show \
  --resource-group "$RG" \
  --name datacenter-vm \
  --show-details \
  --output table
```

Confirm that the VM is:

```text
Running
```

### Retrieve the Public IP

Get the VM's Public IP address:

```bash
PUBLIC_IP=$(az vm show \
  --resource-group "$RG" \
  --name datacenter-vm \
  --show-details \
  --query publicIps \
  --output tsv)
```

Verify the value:

```bash
echo "$PUBLIC_IP"
```

### Verify HTTP Connectivity

Test Nginx externally:

```bash
curl -v "http://$PUBLIC_IP"
```

A successful response should include:

```text
HTTP/1.1 200 OK
Server: nginx
```

and return the default:

```text
Welcome to nginx!
```

### Verify Nginx on the VM

SSH into the VM:

```bash
ssh azureuser@"$PUBLIC_IP"
```

Check the Nginx service:

```bash
systemctl status nginx
```

The expected state is:

```text
active (running)
```

Exit the VM:

```bash
exit
```

Finally, run the KodeKloud challenge validation.

## Result

The Azure VM was successfully provisioned and automatically configured using the Azure CLI:

```text
Azure CLI
   │
   ├── Create datacenter-vm
   │
   ├── Ubuntu image
   │
   ├── Startup script
   │      └── Install + start Nginx
   │
   └── NSG
          └── Allow TCP/80
                 │
                 ▼
             Internet
```

The VM is running Nginx and is accessible from the internet over HTTP port `80`.

## Key Takeaways

- `az vm create` allows an Azure VM and its supporting infrastructure to be provisioned directly from the command line.
- `--custom-data` provides startup configuration that can be processed during the VM's initial provisioning.
- Startup automation removes the need to manually SSH into every server to perform repetitive installation tasks.
- `az vm open-port` can create an NSG rule permitting required inbound traffic.
- Installing Nginx is only one part of the dependency path; the NSG must also allow TCP/80 for external clients to reach it.
- `systemctl enable nginx` ensures Nginx starts automatically after future reboots, while `systemctl start nginx` starts it immediately.
- A successful VM deployment does not prove that the application is functioning.
- An external `curl http://<PUBLIC-IP>` test validates the complete path from the client through Azure networking to the Nginx application.
- Automating provisioning through CLI and startup scripts makes infrastructure deployments more repeatable and reduces manual configuration.

## Status

**Completed Successfully ✅**
