# ansible_for_vault
This is a place to experiment and share ideas for integrating Ansible and Vault.

## Working with this repo
In VS Code, install the Ansible add-on.
Set the language for the playbook/yaml files to "Ansible" rather than "yaml".
You may find it helpful to install ansible-dev-tools with `pip install ansible-dev-tools`.


## Architecture Diagram

+----------------+         (1) Authenticate & (2) Request Password         +----------------+
|                |------------------------------------------------------->|                |
|    Ansible     |                                                        |     Vault      |
|  Playbook/AAP  |<-------------------(3) Return New Password-------------|                |
|                |------(4) Write new password to Vault (KV v2) --------->|                |
+----------------+                                                        +----------------+
        |                                                                         
        | (5) Update break-glass account password                                 
        v                                                                        
+----------------+                                                               
|                |                                                               
| Remote Linux   |                                                               
|   System       |                                                               
| (break-glass   |                                                               
|   account)     |                                                               
+----------------+                                                                

## Reset local Administrator password on Windows host (break glass credential rotation)

* Vault can generate a password according to a password policy.
* Vault can audit who accesses a break glass secret, and when.
* Vault can set/track TTLs for passwords.

* Ansible can connect to Vault and to the Windows remote machine.
* Ansible can reset the user's password.
* Ansible can change the expiration settings for a user's password.

### Prerequisites

* Ansible needs an authentication mechanism for Vault, such as a token or a platform identity trust. A custom credential type may be created, if necessary. This credential will need to be rotated and updated in Ansible within its TTL.
* Ansible needs authorization in Vault policies to read and write the paths where it should manage passwords.
* Ansible needs authentication and authorization on the remote machines to log in and change the local Administrator password. In this case, a domain administrator account will be used by Ansible to gain this access. The domain Administrator account is stored in the AAP password store.
* Ansible needs to have network connectivity to the remote machine on its standard auth/remote access port, which depends on the connection mechanism being used (SSH, WinRM, NTLM, RDP, etc).
* Ansible needs to have network connectivity to Vault on its API port (default Vault API port is 8200).
* The maintainer of the orchestration code needs to understand how operating system permissions are meant to be managed.

### Proposed rotation sequence if Ansible uses/stores a different credential (such as domain admin)

1. Ansible logs in to the remote machine with domain admin credential
2. Verify current password matches:

    a. Ansible logs into Vault and queries current password.

    b. Ansible checks if current password on the remote machine matches the entry in Vault.

    c. If the password does not match, alert.
3. Ansible uses the Vault API or a HashiCorp Vault module to generate a password using a password policy. The password is returned to Ansible.
4. Ansible uses a HashiCorp Vault module or makes an API call to Vault to write the password into an appropriate secrets engine. At this time, the best option is kv v2. In the future or for other types of credentials, there may be different secrets engines that support more streamlined workflows.
5. With the password in memory, Ansible reaches out to the managed remote machine to do the following:

   a. Change the password for the local break glass Administrator account.

   b. Update account aging parameters (if needed)

### Threat model

* The password will pass through Ansible's temporary execution environment.
* Ansible's access to Vault will have rights to change these passwords. The credential Ansible is using to access Vault should be protected as a highly sensitive credential.
* Ansible will log the variables used to retrieve and set the password, unless action is taken to prevent standard logging, such as using `no_log: true` on each task that involves a password.
* Significant alerting should be used when this job fails, since its results are relied upon in emergencies.
* Contributions to the code for this job should be reviewed carefully, and not auto-merged without review. Anyone who can contribute to this playbook can change the behavior in Vault and on the endpoint, and could explicitly set a password other than one retrieved from Vault.
* If your credentials expire, they may lock you out.

### Break glass account lifecycle

Break glass accounts are for emergencies.

Because you cannot predict who will be available to access them in an emergency, **break glass accounts aren't tied to a specific individual.** You need them to be accessible in an emergency.

Break glass accounts should be local administrators on the respective machines, with full access.

Break glass accounts **should not be federated.** This will ensure they are still accessible even in the case of centralized domain auth failure (such as network outages or certificate failures due to a system clock out of sync).

Break glass accounts should not permanently lock out.

Break glass account activity should be monitored, logged, and have noisy notifications for any usage.
