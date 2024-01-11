# caas-ipa-appliance

## Initial setup and deploy

This is a quick guide to using the appliance

first steps are to clone the repo and the copy the test environment ready for your new deployment here I'm using ci-prod

```
git clone git@github.com:eschercloudai/caas-ipa-appliance.git
cd caas-ipa-appliance/
git submodule update --init
cd environments/
cp -a test ci-prod
mkdir ../venv
python3.8 -m venv ../venv/ipaas
source ../venv/ipaas/bin/activate
pip install -U upgrade pip
pip install -r requirements.txt
```

in the above we source the venv we populated but we also need to source the vars for this deployment `environments/ci-prod/activate` (remember to user the name of your environment) and our openstack creds  be this a clouds.yaml or openrc format here I use the openrc for this user and project

```
(ipaas) [cloud-user@control ~]$ cd caas-ipa-appliance
(ipaas) [cloud-user@control caas-ipa-appliance]$ source environments/ci-prod/activate
Setting APPLIANCES_ENVIRONMENT_ROOT to $HOME/caas-ipa-appliance/environments/ci-prod
Setting APPLIANCES_REPO_ROOT to $HOME/caas-ipa-appliance/vendor/eschercloudai/ansible-freeipa-appliance
Setting ANSIBLE_CONFIG to $HOME/caas-ipa-appliance/environments/ci-prod/ansible.cfg
ci-prod/ (ipaas) [cloud-user@control caas-ipa-appliance]$ source ~/openrc-ohpc.sh
Please enter your OpenStack Password for project OHPC as user testuser:
```

after this we can install the requirements using ansible-galaxy like so:

```
ansible-galaxy install -r requirements.yml
```

and then customise the environment

```
(ipaas) [cloud-user@control caas-ipa-appliance]$ diff environments/test/ipa_extra_vars.yml environments/ci-prod/ipa_extra_vars.yml
10,11c10,11
< ipa_cluster_name: "test"
< ipa_cluster_suffix: "DSkjBfsdk.iNTERNAL" # mixed case is just for the testing. this is NOT best practice in any way
---
> ipa_cluster_name: "ci"
> ipa_cluster_suffix: "ci.internal" # mixed case is just for the testing. this is NOT best practice in any way
16,18c16,18
< ipa_cluster_internal_network: "antony-routable" # use this precreated vlan provider net so we can have ironic nodes
< ipa_cluster_internal_subnet: "antony_routable_subnet" # the subnet for that netowrk to avoid extra TF hoops to jump through
< ipa_replica_count: 2
---
> ipa_cluster_internal_network: "slurm-net" # use this precreated net we are in to avoid needing a pointless floating ip
> ipa_cluster_internal_subnet: "slurm-subnet" # the subnet for that netowrk to avoid extra TF hoops to jump through
> ipa_replica_count: 0
```

finally run from `[caas-ipa-appliance](https://github.com/eschercloudai/caas-ipa-appliance)`  dir

```
caas-ipa-appliance]$ ansible-playbook -e @${APPLIANCES_ENVIRONMENT_ROOT}/ipa_extra_vars.yml   ipa-infra.yml
```

When this has completed you will have a functional freeipa (instance or cluster)

this is 100% default with just the IPA server(s) in the DNS and an admin user. The admin user password is templated in plaintext into ${APPLIANCES_ENVIRONMENT_ROOT}/passwords/ipa_admin_password for now.

## post-deploy configuration

### access the IPA CLI/UI

to access the FreeIPA instance we have created using the above defaults we will need to ssh in using a VM hosted in the same network. Assuming that you are are using the same hpc bastion node in the same internal net as the target cluster and that you are not using a floating IP for the freeIPA instance(s) you will need to find the ip of the instance using `openstack server list`. The ip created in this example was 192.168.129. To make it easier it has been added to `/etc/hosts` on the bastion node (this also allows you to use this host as a ssh SOCKS proxy to access the webui which it REALLY handy) like this: 

```
[cloud-user@control ~]$ grep ci.internal /etc/hosts
192.168.99.129 ci-ipa-1.ci.internal
```
it is then possible to ssh in using the hostname or the IP, here is me using the IP in case you don't want to add the hostname. Note that if you have a differnet image with a different cloud-init user, use that rather than rocky

```
[cloud-user@control ~]$ ssh rocky@192.168.99.129
The authenticity of host '192.168.99.129 (192.168.99.129)' can't be established.
ECDSA key fingerprint is SHA256:YgvurGBVjqDV691DIHjkXIAglmXQqby1XPkTvCqjHC0.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.99.129' (ECDSA) to the list of known hosts.
[rocky@ci-ipa-1 ~]$
```

freeipa is running in a podman container as root so get the name which could be freeipa-primary, freeipa-replica or simply freeipa-server run `podman ps` as root like this:
```
[rocky@ci-ipa-1 ~]$ sudo su -
[root@ci-ipa-1 ~]# podman ps
CONTAINER ID  IMAGE                                     COMMAND               CREATED      STATUS      PORTS       NAMES
a0bae951eaa3  docker.io/freeipa/freeipa-server:rocky-9  ipa-server-instal...  6 weeks ago  Up 6 weeks              freeipa-primary
```

get into the podman container `podman exec -it <ipa-container-name> name bash`, authenticate as the admin user `kinit admin` (using the password in the file mentioned above) and list all users in the realm `ipa user-find` using the following:

```
[root@ci-ipa-1 ~]# podman exec -it freeipa-primary bash
[root@ci-ipa-1 /]# kinit admin
Password for admin@CI.INTERNAL:
[root@ci-ipa-1 /]# ipa user-find
--------------
1 user matched
--------------
  User login: admin
  Last name: Administrator
  Home directory: /home/admin
  Login shell: /bin/bash
  Principal alias: admin@CI.INTERNAL, root@CI.INTERNAL
  UID: 1197400000
  GID: 1197400000
  Account disabled: False
----------------------------
Number of entries returned 1
----------------------------
```

if you have 

1. added the hostname to /etc/hosts on your bastion 
2. configured a ssh socks5 proxy on a local port (I used 8076) e.g. ssh -D8076 user@bastion-host
3. pointed your browser at the proxy
   1. in firefox this is found int he main menu -> settings->Configure Network settings (right down the bottom)
   2. click Manual Proxy configuration, add localhost and port number you specifed for the SOCKS Host
   3. click SOCKS v5
   4. no NOT forget to click the Proxy DNS when using SOCKS v5 box

you can point that broswer to https://ci-ipa-1.ci.internal/ipa/ui

note that https:/ and the /ipa/ui need to be typed to facilitate the later addition of  a load balancer later the default redirects in the freeipa-server containers break this functionality so are disabled the hostname should be the fqdn of the freeipa host created by your ipaaas ansible. the hostname is normally [ipa_cluster_name]-ipa-#.[ipa_cluster_suffix] using the variables defined in ipa_extra_vars

# configure for a the caas-slurm-appliance

You should be in the UI/CLI if not follow the above steps

the caas appliance defaults require 3 users:

- ansible_host_service - to add manchines to the DNS/realm
- ldapsearch_user - to allow the dex component of the ood portal access to bind to the ldap directory
- hpc-ipa a user for the customer to use (can be admin)

the first 2 do not need to be able to login  to the cluster so they should not be in the ipausers group and will need custom roles to limit access

note that we

```
[root@ci-ipa-1 /]# ipa user-add ansible_host_service --first=ansible --last=host_add --password --password-expiration=20340111105304Z
Password:
Enter Password again to verify:
---------------------------------
Added user "ansible_host_service"
---------------------------------
  User login: ansible_host_service
  First name: ansible
  Last name: host_add
  Full name: ansible host_add
  Display name: ansible host_add
  Initials: ah
  Home directory: /home/ansible_host_service
  GECOS: ansible host_add
  Login shell: /bin/sh
  Principal name: ansible_host_service@CI.INTERNAL
  Principal alias: ansible_host_service@CI.INTERNAL
  User password expiration: 20340111105304Z
  Email address: ansible_host_service@ci.internal
  UID: 1197400010
  GID: 1197400010
  Password: True
  Member of groups: ipausers
  Kerberos keys available: True
[root@ci-ipa-1 /]# ipa user-add ldapsearch_user --first=ansible --last=host_add --password --password-expiration=20340111105304Z
Password:
Enter Password again to verify:
----------------------------
Added user "ldapsearch_user"
----------------------------
  User login: ldapsearch_user
  First name: ansible
  Last name: host_add
  Full name: ansible host_add
  Display name: ansible host_add
  Initials: ah
  Home directory: /home/ldapsearch_user
  GECOS: ansible host_add
  Login shell: /bin/sh
  Principal name: ldapsearch_user@CI.INTERNAL
  Principal alias: ldapsearch_user@CI.INTERNAL
  User password expiration: 20340111105304Z
  Email address: ldapsearch_user@ci.internal
  UID: 1197400011
  GID: 1197400011
  Password: True
  Member of groups: ipausers
  Kerberos keys available: True
[root@ci-ipa-1 /]# ipa user-add hpc-ipa --first=hpc --last=user --password --shell=/bin/bash --password-expiration=20340111105304Z
Password:
Enter Password again to verify:
--------------------
Added user "hpc-ipa"
--------------------
  User login: hpc-ipa
  First name: hpc
  Last name: user
  Full name: hpc user
  Display name: hpc user
  Initials: hu
  Home directory: /home/hpc-ipa
  GECOS: hpc user
  Login shell: /bin/bash
  Principal name: hpc-ipa@CI.INTERNAL
  Principal alias: hpc-ipa@CI.INTERNAL
  User password expiration: 20340111105304Z
  Email address: hpc-ipa@ci.internal
  UID: 1197400012
  GID: 1197400012
  Password: True
  Member of groups: ipausers
  Kerberos keys available: True
[root@ci-ipa-1 /]# ipa group-remove-member ipausers --users=ldapsearch_user --users=ansible_host_service
  Group name: ipausers
  Description: Default group for all users
  Member users: hpc-ipa
---------------------------
Number of members removed 2
---------------------------
[root@ci-ipa-1 /]# ipa group-add-member service --users=ldapsearch_user --users=ansible_host_service
  Group name: service
  GID: 1197400003
  Member users: ldapsearch_user, ansible_host_service
-------------------------
Number of members added 2
-------------------------
[root@ci-ipa-1 /]# ipa role-find
---------------
8 roles matched
---------------
  Role name: Enrollment Administrator
  Description: Enrollment Administrator responsible for client(host) enrollment

  Role name: helpdesk
  Description: Helpdesk

  Role name: host admin

  Role name: IT Security Specialist
  Description: IT Security Specialist

  Role name: IT Specialist
  Description: IT Specialist

  Role name: Security Architect
  Description: Security Architect

  Role name: Subordinate ID Selfservice User
  Description: User that can self-request subordiante ids

  Role name: User Administrator
  Description: Responsible for creating Users and Groups
----------------------------
Number of entries returned 8
----------------------------
[root@ci-ipa-1 /]# ipa role-show "host admin"
  Role name: host admin
  Member users: ansible_host_service
  Privileges: Host Administrators, Host Group Administrators, DNS Servers, Host Enrollment


```

