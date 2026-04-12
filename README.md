# Overview
 This repository contains the source code for the "EpicBook" application alongside the Ansible playbooks and roles required for its configuration and deployment. This repository focuses on "Phase 2" of the application lifecycle: connecting to infrastructure already provisioned by the [Epicbook-infrastructure repository](https://github.com/Thormie-Harshey/infra-epicbook.git), configuring the application environment (Nginx, Python, dependencies), setting up backend connectivity (MySQL), and deploying the codebase. 
 
 The configuration management and deployment is fully automated via an Azure DevOps YAML pipeline (`azure-pipelines.yaml`) triggered on changes to the `main` branch.
  ## Repository Folder Structure
```text
epicbook/
├── ansible/
│ ├── group_vars/
│ │ └── web/
│ │ ├── vars.yml
│ │ └── vault.yml
│ └── roles/
│ ├── common/
│ │ ├── handlers/
│ │ │ └── main.yml
│ │ └── tasks/
│ │ └── main.yml
│ ├── epicbook/
│ │ ├── tasks/
│ │ │ └── main.yml
│ │ └── templates/
│ │ └── config.json.j2
│ └── nginx/
│ ├── handlers/
│ │ └── main.yml
│ ├── tasks/
│ │ └── main.yml
│ └── templates/
├── .gitignore
├── .vault_pass
├── azure-pipelines.yml
├── inventory.ini
├── README.md
└── site.yml

```
   ## Requirements & Prerequisites 
The azure-pipelines.yaml is procedural and relies heavily on Azure DevOps features for security. Ensure the following prerequisites are met: 
#### 1. Azure DevOps Pipeline Library Configuration
####  Variable Group 
Create a Variable Group named ansible-secrets in Pipelines > Library.
-  Add the required secrets, most importantly: ANSIBLE_VAULT_PASSWORD. This password must match the one used to encrypt ansible/group_vars/web/vault.yml. 
#### Secure Files 
- Ensure you have uploaded the private SSH key (id_rsa) to the Secure Files section of your pipeline library. This key must correspond to the public key injected by the infrastructure pipeline.
#### Agent Pool
The pipeline is configured to use a self-hosted agent named epicbook-agent-pool. Ensure your agent is configured and online. 
### 2. Static Application Variables (For Now) 
Currently, variables such as the VM Public IP and Database FQDN are managed as static variables within the pipeline YAML (azure-pipelines.yaml). Before running the pipeline, update these variables with the outputs from your Infrastructure (Terraform) deployment: 
```YAML
variables:
  - group: ansible-secrets
  - name: VM_IP
    value: 'UPDATE_THIS_WITH_YOUR_PUBLIC_IP'
  - name: DB_FQDN
    value: 'UPDATE_THIS_WITH_YOUR_DATABASE_HOST_URL'
```
*Note: In future iterations, this workflow can be automated to pass variables directly from the Terraform pipeline outputs.*

### Usage
 Any commit pushed to the main branch will automatically trigger the azure-pipelines.yaml. 
### Pipeline Execution Workflow: 
1. **Preparation**: Downloads the private SSH key securely from Secure Files and creates a temporary .vault_pass file on the agent's disk using the linked variable group secret.
2.  **Dynamic Inventory Generation**: A script dynamically creates the ansible/inventory.ini file using the pipeline variables (VM_IP), targeting the azureuser. 
3. **Ansible Execution**: Runs the ansible-playbook site.yml command, leveraging the generated inventory, the secure private key, the vault password file, and passing the database host as an extra variable.
4.  **Cleanup (CRITICAL Best Practice)**: A final step ensures the temporary .vault_pass file is deleted from the agent's disk, even if the deployment failed. This is handled by condition: always().
### Verification
Once Pipeline 2 completes successfully, you can verify the deployment by navigating to the Nginx public IP address in your browser. The EpicBook application should load, successfully querying the backend database.

### Watch out for this:
**1. Missing Binaries on Self-Hosted Agents**
 Unlike Microsoft-hosted agents which come "pre-loaded" with almost everything, a self-hosted agent is a blank canvas. If your pipeline calls `terraform` or `ansible-playbook` and the tool isn't in the system's `$PATH`, the job will fail with a "command not found" error.

**The Fix:** You have two choices:

-   **Pre-install:** Manually install Terraform and Ansible on the VM hosting your agent.
-   **On-the-fly:** Use the `TerraformInstaller@1` or `Pip` tasks at the start of your pipeline to ensure the correct version is present every single time.

**2. SSH Fingerprint Prompt (The Silent Killer)**

When Ansible tries to connect to a new Azure VM for the first time, SSH usually pauses and asks: _"The authenticity of host... can't be established. Are you sure you want to continue connecting (yes/no)?"_ In a pipeline, there is no one to type "yes," so the pipeline just hangs until it times out.

**The Fix:** Use the environment variable `export ANSIBLE_HOST_KEY_CHECKING=False` in your script block. This tells Ansible to skip that manual verification and proceed with the automation.