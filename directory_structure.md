# Create the Directory Structure

This lab uses two network adapters on both the controller and SAP server to separate SAP traffic from internet access (needed for collection installation).

---
```
# Create the full project directory tree in one command
mkdir -p ~/sap-ansible/{inventory/group_vars/{all,sap_servers},playbooks,roles,collections}

# Verify structure was created correctly
find ~/sap-ansible -type d | sort
```

## Expected Output:

<img width="411" height="135" alt="image" src="https://github.com/user-attachments/assets/f7fe0472-49d2-427f-9c1b-e3a081cbbecd" />

---

## Create the Inventory File

An inventory is the answer to one question: "What servers does Ansible know about, and how are they organized?"

But it is more than a list of hostnames. An inventory defines:
```
WHAT the inventory actually contains:
─────────────────────────────────────
1. HOST DEFINITIONS    —> individual servers + connection details
2. GROUPS              —> logical groupings of hosts
3. GROUP HIERARCHY     —> groups that contain other groups
4. HOST VARIABLES      —> variables specific to one host
5. GROUP VARIABLES     —> variables applied to all hosts in a group

```

### Create group_vars directory structure
```
mkdir -p inventory/group_vars/{all,sap_servers,sap_db,sap_app}
```

1. HOST DEFINITIONS

    Create [hosts.yml](https://github.com/ramawahyuk/ansible-sap-rhel-automation-lab/blob/main/hosts.yml) inside ~/sap-ansible/inventory 

```
cd ~/sap-ansible
vi inventory/hosts.yml

```
2. Common Variables

    create [variables](https://github.com/ramawahyuk/ansible-sap-rhel-automation-lab/blob/main/common.yml) that apply to all hosts:

```
#create all-host common variables
vi inventory/group_vars/all/common.yml
```
 3. SAP server group variables

      Create the [variable](https://github.com/ramawahyuk/ansible-sap-rhel-automation-lab/blob/main/sap_common.yml) files that will hold SAP-specific settings:
```
vi inventory/group_vars/sap_servers/sap_common.yml

```
4. Oracle DB specific variables

    create the [variable](https://github.com/ramawahyuk/ansible-sap-rhel-automation-lab/blob/main/oracle.yml) files of Oracle for SAP

```
vi inventory/group_vars/sap_db/oracle.yml
```
###Verify Inventory Structure

```
cd ~/sap-ansible

# List all files created
find inventory/ -type f | sort
```
 

<img width="485" height="126" alt="image" src="https://github.com/user-attachments/assets/f1c3b704-82ad-4e56-ae3b-d38929bdddcd" />

## 

```
# Verify Ansible reads the inventory correctly
ansible --list-hosts all
```


<img width="450" height="62" alt="image" src="https://github.com/user-attachments/assets/c3388a45-f5ca-4b97-94a3-d2d5f7fc3322" />


##

```
# Verify group membership
ansible --list-hosts sap_servers
ansible --list-hosts sap_db
ansible --list-hosts sap_app

```


<img width="502" height="181" alt="image" src="https://github.com/user-attachments/assets/a3532065-e74e-49ac-9b2c-6a6d81f5fabc" />


##

## Create the ansible.cfg File 
```
┌─────────────────────────────────────────────────────────┐
│                   ansnode ~/sap-ansible/                │
│                                                         │
│  ansible.cfg ───────────────────────────────────────┐   │
│  (the compass)                                      │   │
│       │                                             │   │
│       ├──► inventory/hosts.yml     (WHO)            │   │
│       ├──► inventory/group_vars/   (VARIABLES)      │   │
│       ├──► playbooks/              (WHAT TO DO)     │   │
│       ├──► roles/                  (HOW TO DO IT)   │   │
│       └──► collections/            (COLLECTION LIB) │   │
│                                                     │   │
│  When you run:                                      │   │
│  ansible-playbook playbooks/hana_install.yml        │   │
│       │                                             │   │
│       └── Ansible reads ansible.cfg first ──────────┘   │
│           Then loads inventory, vars, collections       │
└─────────────────────────────────────────────────────────┘
```

[ansible.cfg](https://github.com/ramawahyuk/ansible-sap-rhel-automation-lab/blob/main/ansible.cfg) tells Ansible: "look for roles here, look for collections here, use this inventory by default." Without ansible.cfg, Ansible searches system-wide paths.

validate ansible.cfg is recognized in our system. Go to ~/sap-ansible then:

```
# Confirm Ansible picks up our new config (not /etc/ansible/ansible.cfg)
ansible --version | grep "config file"

```
Expected Output:

<img width="543" height="44" alt="image" src="https://github.com/user-attachments/assets/33a789a1-d634-41c3-aebd-ad650590ae13" />

## 

If it still shows None or /etc/ansible/ansible.cfg, you are not in the right directory. It can be seen that we dont need to use -i inventory/hosts.yml again, just call sap_servers we can check the connection between two nodes.

<img width="467" height="92" alt="image" src="https://github.com/user-attachments/assets/08c810e4-6aa5-4e2b-aaee-00b3d0c017c8" />


---

## Verify the full structure of directory which has files in it

```bash
find ~/sap-ansible -type f | sort
```

<img width="535" height="197" alt="image" src="https://github.com/user-attachments/assets/baba9ff2-06ad-4f48-9fb8-d5479b464772" />






