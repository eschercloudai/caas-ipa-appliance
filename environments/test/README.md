# FreeIPA as a service test env

first go

```
source test/activate

# Configure
ansible-playbook -e @${APPLIANCES_ENVIRONMENT_ROOT}/ipa_extra_vars.yml ${APPLIANCES_ENVIRONMENT_ROOT}/../../ipa-infra.yml

# Tear down
ansible-playbook -e @${APPLIANCES_ENVIRONMENT_ROOT}/cluster_extra_vars.yml -e cluster_state=absent ${APPLIANCES_ENVIRONMENT_ROOT}/../../ipa-infra.yml
```