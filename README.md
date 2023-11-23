# caas-ipa-appliance

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
> ipa_cluster_internal_network: "slurm-net" # use this precreated vlan provider net so we can have ironic nodes
> ipa_cluster_internal_subnet: "slurm-subnet" # the subnet for that netowrk to avoid extra TF hoops to jump through
> ipa_replica_count: 0
```

finally run from `[caas-ipa-appliance](https://github.com/eschercloudai/caas-ipa-appliance)`  dir

```
caas-ipa-appliance]$ ansible-playbook -e @${APPLIANCES_ENVIRONMENT_ROOT}/ipa_extra_vars.yml   ipa-infra.yml
```

