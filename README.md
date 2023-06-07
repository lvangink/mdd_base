# Model-Driven DevOps Repo

This is the base repo for Model-Driven DevOps.  It has the needed python requirements (requirements.txt) and
Ansible Collection (requirements.yml) to run a Model-Driven DevOps pipeline.  It requires an ansible inventory and
certain environment variables to be set. It also assumes you are using Cisco Modeling Labs, GitLab, and NSO. Details on running
NSO and CML can be found by using the following links.


# Installing Tooling
## NSO
Installing in Kubernetes  
Installing in CML  
Installing in Docker  
Installing Locally  

## CML
Installing in AWS  
Installing in VMware  

## GitLab

# Installation
## Clone Repo Template
To use this template, you can select the "Use this template" option from the GitHub UI if your MDD repo will be created in GitHub. If you intend to use GitLab, you can create a new repo from this template manually.

First clone the template locally.
```
git clone git@github.com:model-driven-devops/lvangink/mdd-base.git <your-repo-name>
```
Move to the new repo.
```
cd <your-repo-name>
```
Remove the git folder.
```
rm -rf .git
```
Initialize your new git repo and commit the repo contents.
```
git init .
git add .
git commit -m "Initialize new repo"
```
Create a new repo in your target env using your method of choice and then set that as the repo's remote.
```
git remote add origin https://<your-git>/<your-repo>.git
git push -u -f origin main
```
Note: These instructions and the CI file assume that your git is configured to use "main" as the primary branch. You can configure this to be the default behavior by running `git config --global init.defaultBranch main`

## Configure Repository
A commented [sample CI file](.gitlab-ci.yml) has been provided to demonstrate a recommended CI pipeline. Once NSO, CML, and the inventories have been configured and network configs have been harvested, the pipeline in that file can be uncommented to run the pipeline as-is, or modified to suit your needs. 

## Installing Dependencies

### Python 
Next, it is highly recommended that you create a virtual environment to make it easier to install the dependencies without conflict:
```
python3 -m venv venv-mdd
. ./venv-mdd/bin/activate
```
Next, install the Python requirements via pip:
```
pip3 install -r requirements.txt
```

### Reactivate Virtual Environment
Reactivate virtual environment to ensure your shell is using the newly installed ansible.
```
deactivate
```
```
. ./venv-mdd/bin/activate
```
### Environmental Variables
The MDD tooling requires several environment variables. The first one required for base execution is:
```
export ANSIBLE_PYTHON_INTERPRETER=${VIRTUAL_ENV}/bin/python
export ANSIBLE_COLLECTIONS_PATH=./
```
### Ansible Collections
```
ansible-galaxy collection install -r requirements.yml
```

### Environment Variables
Set the environment variables listed below:

#### NSO
The TEST_REMOTE and PROD_REMOTE passwords are used for your device authentication.
```
export NSO_USERNAME=
export NSO_PASSWORD=
export TEST_REMOTE_PASSWORD=
export PROD_REMOTE_PASSWORD=
```

#### CML
```
export CML_HOST=cml.domain.com
export CML_USERNAME=user
export CML_PASSWORD=password
export CML_LAB=mdd
export CML_VERIFY_CERT=false
```

# Setup Your Inventory
The base repository contains two ansible inventory directories. The directory labeled inventory_prod will contain an nso.yml file that points to the NSO instance managing your production devices. Below is an example of how your nso.yml file should look.

### inventory_prod/nso.yml
```
 nso_rest_url: http://xx.xx.xx.xx:8080
    nso_username: admin
    nso_auth_groups: default
      default:
        remote_name: admin
        remote_password: "{{ lookup('env', 'PROD_REMOTE_PASSWORD') | default('admin', true) }}"        
  children:
    nso:
      vars:
        ansible_user: "{{ lookup('env', 'NSO_USERNAME') | default('ubuntu', true) }}"
        ansible_password: "{{ lookup('env', 'NSO_PASSWORD') | default('admin', true) }}"
        ansible_python_interpreter: /usr/bin/python3
      hosts:
        nso1: 
          ansible_host: xx.xx.xx.xx:8080
```

Follow the same instructions for the nso.yml file in your inventory_test directory with the information for your test instance of NSO.

### inventory_prod/network.yml
The network.yml file is the key element for your NetDevOps pipeline. It defines your infrastructures hierarchy and design. It also allows you to provide tags to specific devices, giving operators the ability to execute specific tests or push modular pieces of configuration to a subset of your infrastructure. The base repository provides a sample network.yml inventory file that can be used as a template to build your network. This template is taken from the core MDD repository and maps to the exercises that can be executed against our reference topology.

When implementing MDD in a brownfield environment, we want to harvest the configurations from our production network, so it is recommended to start with the network.yml file in your inventory_prod folder.

If you are using this base repository with CML and you do not have a physical environment, you do not need to provide any additional information for each device. The ansible collection takes advantage of the CML API to automatically grab the IP addresses for each device.
```
  children:
    network:
      children:
        switches:
          hosts:
            hq-sw1:
            hq-sw2:
            site1-sw1:
            site2-sw1:
```

If you are harvesting your data from a physical environment, you will need to provide IP addresses for each device as depicted in the example below:

```
  children:
    network:
      children:
        switches:
          hosts:
            hq-sw1:
              ansible_host: 10.0.0.1
            hq-sw2:
              ansible_host: 10.0.0.2
            site1-sw1:
              ansible_host: 10.0.0.3
            site2-sw1:
              ansible_host: 10.0.0.4
  ```
## Harvesting Data
Now that our inventories have been created, we need to update our production instance of NSO with our inventory. Start out by adding the default auth group to NSO so it can connect to your devices.

```
ansible-playbook ciscops.mdd.nso_init -i=inventory_prod
```
Now that our auth group has been created, we can have NSO connect to the devices by running the following playbook:

```
ansible-playbook ciscops.mdd.nso_update_devices -i=inventory_prod
```

Once the devices from the network.yml file in your inventory_prod directory have been added, we can execute a sync action for NSO to ingest the configurations into its database.

```
ansible-playbook ciscops.mdd.nso_sync_from -i=inventory_prod
```
Next, we will harvest the configurations as structured data in the OpenConfig format. If you would like to harvest the native data, refer to the additional commands section of this guide.

```
ansible-playbook ciscops.mdd.harvest -i=inventory_prod
```

You should start to see the configurations being populated under the mdd-data directory matching the hierarchy defined in your network.yml file.

![Screenshot 2023-06-05 at 4 37 14 PM](https://github.com/lvangink/mdd_base/assets/65776483/e01fbc26-cf02-49c3-8460-39b03f7f28da)

You will see files parsed as both oc-feature.yml and config-remaining.yml. We are regularly adding supported OpenConfig features to the NSO service. While we continue to add support, you will have your data represented as both OpenConfig and Native.

## Building Your Digital Twin
If you do not have an existing simulation environment for pipeline testing, our ansible collection includes a playbook that will use CDP and LLDP to map your network and build a simulation that can be used to import into CML. This playbook also builds the interface mapping table necessary to ensure your simulation matches your production interface naming convention.

Before you run the simulation creation playbook, navigate to your group_vars directory under your inventory_prod directory and you will see a cml.yml file. This file enables the use of the virtual catalyst 9000 switch for features specific to layer 3 switches. If you require L3 switching, uncomment the following:
```
  l3switch:
    node_definition: Cat9000v
    image_definition: Cat9000v-24p
    ram: 18432
    cpus: 4
    tags:
      - l3switch
    type: l3switch
```
If you do not require L3 switching, you can leave the Cat9000v commented out and make sure the following is uncommented:
```
    l3switch:
      node_definition: iosvl2
      ram: 768
      tags:
        - l3switch
      type: l3switch
```
Now we can execute the playbook with the proper Cat9kv flag set:

```
ansible-playbook ciscops.mdd.cml_update_lab -i inventory_prod -e start_from=1 -e inventory_dir=inventory_test -e use_cat9kv=yes
```
If you are not using the Cat9kv, you can set use_cat9kv=no

Once the playbook executes, you will have a cml.yml file created under the files directory and a cml_intf_map.yml file created under your inventory_test directory.

![Screenshot 2023-06-05 at 5 04 34 PM](https://github.com/lvangink/mdd_base/assets/65776483/15dcf997-6d4b-4c82-bb76-8f3942ec5817)

## Deploy Digitial Twin
Once you have your cml.yml file in your files directory, you can import it into CML by selecing the Import icon on your main screen:

![Screenshot 2023-06-06 at 12 19 01 PM](https://github.com/lvangink/mdd_base/assets/65776483/c926a695-171f-42a9-b3f6-e5966ebdcc9b)

Note: The CML topology creation requires CDP or LLDP turned on to map the network. 

## Setting up your test environment

In our previous steps, we created our production ansible inventory, populated NSO, and harvest the configurations locally. Now that we have our simulated digital twin in CML, we want to set up our test environment so we can create our pipeline.

### inventory_test/network.yml

If you haven't done so already, copy your production network.yml file into your inventory_test directory. Since we are using CML the host names of the devices will remain the same, but you can remove the IP addresses. If you are not using CML, make sure you update the IP addresses for each device before adding them to your test NSO instance.

Once your inventory is completed, we are going to follow the same steps to add the devices to our test NSO but target the inventory_test directory when running our playbooks.

```
ansible-playbook ciscops.mdd.nso_init -i=inventory_test
```
```
ansible-playbook ciscops.mdd.nso_update_devices -i=inventory_test
```
```
ansible-playbook ciscops.mdd.nso_sync_from -i=inventory_test
```
Now our simulated devices are in NSO, we can push the data we harvested from production into our simulation.
```
ansible-playbook ciscops.mdd.update -i=inventory_test
```
By default the update playbook sets the dry_run flag to yes. This sends the data to NSO and verifies that NSO is capable of pushing it to the devices. To have NSO actually push the configuration data, you can set the dry_run flag to no using the following command:
```
ansible-playbook ciscops.mdd.update -i=inventory_test -e dry_run=no
```
