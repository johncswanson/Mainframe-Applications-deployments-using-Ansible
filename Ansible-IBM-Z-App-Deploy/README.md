# Ansible-IBM-Z-App-Deploy
IBM Z App deployment using RedHat Ansible - Sample Playbook

### ansible.cfg 
This contains the required Ansible configuration that needs to be modified as per the environment.
	
### inventory
This contains the details of the managed host where the playbooks will be executed.
	
### deploy_app.yml
Run this Ansible playbook to deploy the application.
	
### rollback_app.yml
Run this Ansible playbook to rollback the application deployment.
	
### ./tasks/*.yml
These are the YAML files for various tasks used in deploy_app.yml and rollback_app.yml

### ./groups_vars/all.yml
This contains the environment variables required for running the Ansible tasks in z/OS.  This needs to be modified according to the environment.

### ./groups_vars/deploy_vars.yml
This contains the application related data required for deployment.  This needs to be modified according to the environment.

### ./files/*.j2
These are the template files for the required JCLs for application deployment.
	
