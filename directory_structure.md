# Create the Directory Structure

This lab uses two network adapters on both the controller and SAP server to separate SAP traffic from internet access (needed for collection installation).

---

# Create the full project directory tree in one command
mkdir -p ~/sap-ansible/{inventory/group_vars/{all,sap_servers},playbooks,roles,collections}

# Verify structure was created correctly
find ~/sap-ansible -type d | sort
---
