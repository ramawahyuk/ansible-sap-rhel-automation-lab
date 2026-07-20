## Screenshots and sample outputs for the verification

## SSH passwordless test

<img width="688" height="113" alt="image" src="https://github.com/user-attachments/assets/f2acdec9-9fb4-4dae-b69b-cc1553170c84" />

## ansible -m ping sap_servers

<img width="461" height="93" alt="image" src="https://github.com/user-attachments/assets/083f3197-e884-48e0-b0d9-bcad122cb1a6" />


## playbooks/vault_test.yml

<img width="1450" height="1085" alt="vault" src="https://github.com/user-attachments/assets/8c131b6b-b9a2-4453-81cd-0cb57af9dd1a" />

## playbooks/os_info.yml
<img width="1026" height="600" alt="image" src="https://github.com/user-attachments/assets/e34bcf61-459d-47ec-bd82-558fd79cf1a1" />

## playbooks/sap_hana_preconfigure.yml
```
ansible-playbooks playbook/sap_hana_preconfigure.yml --check

```
<img width="677" height="650" alt="image" src="https://github.com/user-attachments/assets/e70802af-d5ec-4ce4-9ca9-74a7b4d37f46" />

> This is not an error but the system requires to be rebooted, wait for the host to come back up, then re-run the same playbook to confirm it completes clean and idempotently it should mostly show ok/skipping now, with no more changed on the items already applied

## playbooks/hana_install.yml --tags=sap_hana_install_hdblcm_commandline

Dry-run to confirm clean HANA DB file detection

<img width="975" height="1046" alt="image" src="https://github.com/user-attachments/assets/e3fcc808-b125-46c6-879c-54baa32dc4b0" />

## playbooks/hana_install.yml
Once every check are done, run the real installation

<img width="975" height="694" alt="image" src="https://github.com/user-attachments/assets/5084fc40-cfb2-49dd-b737-2fb3f7c2d139" />
<img width="975" height="476" alt="image" src="https://github.com/user-attachments/assets/6649eb88-31db-4a36-a055-3a074c083891" />

### Try to login into HANA DB using s4dadm on saperp

```
#login into HANA DB administrator
su – s4dadm

#check if all services are running or not
/usr/sap/S4D/SYS/exe/hdb/sapcontrol -prot NI_HTTP -nr 00 -function GetProcessList

```

<img width="975" height="245" alt="image" src="https://github.com/user-attachments/assets/c86dd1e7-1b59-4ea4-991a-cb6d59eb2aab" />

## playbooks/swpm_install.yml

To ensure that the installation are finished check and trace the `sapinst.log` and `sapinst_dev.log`

<img width="975" height="650" alt="image" src="https://github.com/user-attachments/assets/2dc87d3a-060c-4ed7-b382-7e02e36beb3f" />

<img width="975" height="219" alt="image" src="https://github.com/user-attachments/assets/123cbb2b-9259-4677-bdc7-55f09740c480" />

### Validate the running system

```
# On saperp:
su - nw1adm -c "sapcontrol -nr 01 -function GetProcessList"   # ASCS: msg_server, enserver GREEN
su - nw1adm -c "sapcontrol -nr 02 -function GetProcessList"   # PAS: disp+work GREEN
cat /usr/sap/sapservices | grep NW1                            # both instances registered
su - nw1adm -c "hdbuserstore list"                             # DEFAULT key → saperp:30015

#Gather per-instance metadata
su - nw1adm -c "sapcontrol -nr 01 -function GetInstanceProperties" | grep -i "SAPSYSTEMNAME\|INSTANCE_NAME"  
su - nw1adm -c "sapcontrol -nr 02 -function GetInstanceProperties" | grep -i "SAPSYSTEMNAME\|INSTANCE_NAME"
```

<img width="916" height="186" alt="image" src="https://github.com/user-attachments/assets/5375cecb-b114-4f8a-a48f-e8510747a324" />


<img width="975" height="144" alt="image" src="https://github.com/user-attachments/assets/5bf59c3f-a9f1-46bd-915c-0540210cacbb" />


<img width="975" height="389" alt="image" src="https://github.com/user-attachments/assets/73435e20-39e4-4824-b784-bab1c54413d4" />


<img width="975" height="83" alt="image" src="https://github.com/user-attachments/assets/ba52ce1e-8d9e-4524-9d4f-4ded63cf5870" />

## SAP GUI Connection Parameters


<img width="975" height="1005" alt="image" src="https://github.com/user-attachments/assets/57591e9a-6609-4a10-b43f-a0b057cafae4" />

## SAP GUI 


<img width="975" height="520" alt="image" src="https://github.com/user-attachments/assets/2ef397c6-3270-40cb-9490-810e90632211" />
