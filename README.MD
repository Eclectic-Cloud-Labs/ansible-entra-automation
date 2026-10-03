# Ansible Entra Automation Script 
## Creates app registrations, service principals, and client secrets in Microsoft Entra through Graph API.

## Run
ansible-playbook playbooks/graph-auth.yml       # test: lists app regs
ansible-playbook playbooks/create-app-reg.yml   # creates if absent, skips if present

Change "app_display_name" in the create playbook's "vars:" block to create different app names.


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
    - "-curl -X POST -H "Content-Type: application/x-www-form-urlencoded" -d 'client_id={client_id}&scope=https%3A%2F%2Fgraph.microsoft.com%2F.default&client_secret=A1bC2dE3f...&grant_type=client_credentials' 'https://login.microsoftonline.com/{tenant}/oauth2/v2.0/token'"
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
    - ansible-vault encrypt inventory/group_vars/all/vault.yml
11. Create a ".vault_pass" file (gitignored) in the project root with your vault password on a single line, then lock it down with
    - chmod 600 .vault_pass
12. Check if Ansible can read your variables and reach the host
    - ansible localhost -m ping
    - ansible localhost -m debug -a "var=client_id"
13. Run auth playbook to confirm Graph is reachable and permissions are good
    - ansible-playbook playbooks/graph-auth.yml
14. Run the create-app-reg playbook. Change the value of "app_display_name" in the playbook's "vars:" block before running how you want
15. First run creates the app. Second run with the same name reports it already exists and skips all creates.


## Design choices
- Used "ansible.builtin.uri", not "azure.azcollection". Brief was "Graph API calls." and uri keeps the requests visible and no install
- Server side idempotency using OData ("when: existing_apps.json.value | length == 0") guards the create. Same thing for the SPN check.
- Permission scoping started with "Application.ReadWrite.OwnedBy" + "Directory.Read.All" on the automation SP. Had to add "Application.ReadWrite.All" to create service principals. In production I would split responsibilities across two SPs.
- "status_code: 201" on create tasks. Graph returns 201 which was an error after runtime. Created on POST to "/applications" and "/servicePrincipals". Aligned expectation with Graph's semantics to stop a false failure.
- Tenant ID and client ID are in plaintext "vars.yml". Only the client secret is encrypted in "vault.yml". "no_log: true" on the token task so the secret never prints.
- Retry loop on SPN create. "retries: 12", "delay: 5", "until: created_spn is succeeded". Handles Entra's propagation gap between app creation and SPN create without hardcoding a sleep.
- Generated client secret is printed once. "addPassword" returns "secretText" in the response and never again. Playbook prints it with debug task after runtime so the operator can copy it. In production I would write it to Key Vault instead and keep "no_log: true" on the task.


## Incidents
- WSL install hung. First "wsl --install" froze mid setup and every "wsl" command thereafter hung behind it. Full Windows reboot cleared it along with uninstall, then reinstall.
- Vars not loading from "group_vars/all". The playbook couldn't resolve "client_id" even though the file was correct. Debugged by first confirming if the playbook was running at all, then writing a second minimal playbook to check whether the issue was the playbook or the variable itself. Root cause was "group_vars/" not sitting with either the inventory or root, which the playbook doesn't anchor to. Moved it to "inventory/group_vars/" so it resolves off the inventory. Works for both ansible cmds and playbook runs now.
- "Cannot concat str to bytes" on the uri module. Reference: https://liquidat.wordpress.com/2019/08/22/short-tip-handling-cant-concat-str-to-bytes-error-using-the-uri-module/
- "$filter" with raw spaces. URLs can't carry unencoded spaces. Encoded them as "%20" around "eq".
- 201 treated as failure. Graph returns "201 Created" on POST to "/applications" and "/servicePrincipals". Added "status_code: 201" to both create tasks. Reference: https://forum.ansible.com/t/playbook-showing-as-failed-but-actually-runs-successfully/5146/2
- 403 on SPN creation. "Application.ReadWrite.OwnedBy" is enough for the SP to create and manage app registrations it owns, but creating a service principal is tenant wide. Had to replace with "Application.ReadWrite.All" and grant admin consent.
- Quoting rules depend on where they are used. For OData filtering, single quotes are needed for queries ("$filter=appId eq 'guid'"), while URL path segments don't ("/applications/{{var}}/addPassword"). Inserting the quotes into the path produced a string that had extra quotes ("Invalid object identifier ''guid''".)
- SPN creation failing with a new app registration. I kept receiving an error: POST "/servicePrincipals" returned "NoBackingApplicationObject" even though the app registration had just been created successfully in the previous task. Triple checked the appId being passed and it was correct, however, Entra hadn't propagated the new app object and so it wasn't retrievable for when the task to create the SPN was being run. Fixed with "retries: 12", "delay: 5", "until: created_spn is succeeded" on the SPN task, so the playbook waits instead of failing. This is what makes full app + SPN + secret provisioning work in a single run rather than requiring two passes.


## Sources
- How I gave the App Reg its permissions: https://learn.microsoft.com/en-us/graph/api/application-list?view=graph-rest-1.0&tabs=http
- Token for Azure: https://login.microsoftonline.com/common/v2.0/.well-known/openid-configuration (from https://learn.microsoft.com/en-us/entra/identity-platform/access-tokens)
- Actual POST flow: https://stackoverflow.com/questions/75751477/how-to-get-a-microsoft-azure-authorization-token and https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-client-creds-grant-flow
- Get access without a user: https://learn.microsoft.com/en-us/graph/auth-v2-service?tabs=http
- Filtering for the playbook URL: https://learn.microsoft.com/en-us/graph/query-parameters?tabs=http
- How to write OData queries: https://learn.microsoft.com/en-us/azure/devops/report/powerbi/odataquery-connect?view=azure-devops
- Spacing in URLs: https://stackoverflow.com/questions/5442658/spaces-in-urls
- Creating an application: https://learn.microsoft.com/en-us/graph/api/application-post-applications?view=graph-rest-1.0&tabs=http
- GET request for SPN: https://learn.microsoft.com/en-us/graph/api/serviceprincipal-get?view=graph-rest-1.0&tabs=http
- POST request to create SPN: https://learn.microsoft.com/en-us/graph/api/serviceprincipal-post-serviceprincipals
- group_vars placement: https://www.golinuxcloud.com/ansible-group-vars-host-vars-patterns/
