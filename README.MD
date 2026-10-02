# Ansible Entra Automation Script 
## Creates App Registrations on Azure by linking a current active App Registration to the script.


## Stack
- Azure PAYG account (No charge for App Reg's)
- Windows & WSL (Ubuntu) as the Ansible control node
- Ansible core 2.20
- Microsoft Graph API v1.0
- Git + GitHub for version control


## Setup
1. Create App Registration on Azure
    - Single tenant
    - no redirect URL
2. API permissions > Add permission > Add Graph API with type "Application Permissions" (App Reg is used solo) > Add permissions
    - Add these permissions as set out by MS learn - "Application.ReadWrite.OwnedBy" & "Directory.Read.All"
3. Grant Admin consent for Default directory (App doesn't need you to authorize each request however tokens will still be used)
4. Create client secret: Certificates & secrets > New client secret > Enter description, set expiry > add secret, and save the secret value and secret ID somewhere (needed for the script)
5. Test if the App registration works by request a token
    - "-curl -X POST -H "Content-Type: application/x-www-form-urlencoded" -d 'client_id=816ff4a3-965d-4a16-8d6a-18f7d64d3f06&scope=https%3A%2F%2Fgraph.microsoft.com%2F.default& /
      client_secret=A1bC2dE3f...&grant_type=client_credentials' 'https://login.microsoftonline.com/{tenant}/oauth2/v2.0/token'"
6. Install WSL and Ubuntu on Windows if not already done
    - "wsl --install"
7. Inside Ubuntu, install Ansible
    - sudo apt update && sudo apt upgrade -y
    - sudo apt install -y ansible
    - ansible --version
8. Clone the repo
    - git clone https://github.com/Eclectic-Cloud-Labs/ansible-entra-automation.git
    - cd ansible-entra-automation
9. Add tenant ID and client ID to "inventory/group_vars/all/vars.yml"
10. Add client secret to "inventory/group_vars/all/vault.yml", then encrypt it
    - ansible vault encrypt inventory/group_vars/all/vault.yml
11. Create a ".vault_pass" file (gitignored) in the project root with your vault password on a single line, then lock it down with
    - chmod 600 .vault_pass
12. Check if Ansible can read your variables and reach the host
    - ansible localhost -m ping
    - ansible localhost -m debug -a "var=client_id"
13. Run auth playbook to confirm Graph is reachable and permissions are good
    - ansible-playbook playbooks/graph-auth.yml
14. Run the create-app-reg playbook. Change the value of "app_display_name" in the playbook's "vars:" block before running to your fitting 
15. First run creates the app. Second run skips because the app already exists


## Incidents


## Sources


##
