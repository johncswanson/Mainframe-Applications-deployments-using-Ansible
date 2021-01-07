# Simplify mainframe application deployments using Ansible

## Introduction
IBM Z, the new mainframe is the home to mission critical applications all along and continues to be so as it offers unparalleled quality of service including security, availably, reliability and scalability. Pandemic situation worldwide has accelerated digital transformation of enterprises with a surge in adoption of DevOps and hybrid cloud.  IBM has brought DevOps to IBM Z including Continuous Integration (CI), Continuous Deployment (CD) and automated testing.  Like any platform, you can integrate enterprise tools like Git, Jenkins, SonarQube, Artifactory to create a DevOps pipeline for mainframe applications.

Open source based Ansible automation is the latest addition to mainframe DevOps toolchain that can simplify a variety of things including application deployments, middleware provisioning, orchestration and much more. All these can be accomplished using Ansible playbooks, a simple and easy to read script written in YAML that can be version controlled. Ansible employs an agentless architecture, typically consisting of a x86 Linux based control node with a number of managed nodes that span a variety of platforms like Linux/Unix/Windows (LUW) and IBM Z. Ansible runs playbooks on the control node to automate and bring managed nodes to a desired state. With the introduction of [Red Hat® Certified Ansible Content for IBM Z](https://www.ibm.com/support/z-content-solutions/ansible/), you can accelerate your adoption of Ansible for IBM Z to bring in additional efficiencies in your mainframe DevOps pipelines.  

This tutorial discusses how to deploy mainframe applications using Ansible and integrate Ansible playbooks with Jenkins to implement continuous deployment in DevOps pipeline. 


## Prerequisites
- [Ansible](https://www.ansible.com/) 2.9 or later
- [z/OS Core collection](https://galaxy.ansible.com/ibm/ibm_zos_core) for Ansible
- [Python, z/OS OpenSSH and IBM Z Open Automation Utilities](https://ibm.github.io/z_ansible_collections_doc/ibm_zos_core/docs/source/requirements_managed.html) in Managed Node (z/OS)
- Any Git based repositories like [GitHub](https://github.com/), [GitLab](https://about.gitlab.com/), [Bitbucket](https://bitbucket.org/) etc.
- [Jenkins](https://www.jenkins.io/doc/book/getting-started/) orchestrator


## Estimated Time
It should take about 2 hours to complete this tutorial .

## Steps
### IBM Z application deployment using Ansible
For ease of discussion, consider a simple CICS-COBOL-DB2 application. Typical steps to deploy such a mainframe application include: 
- Backup load modules and DBRMs for rollback actions as needed
- Copy updated load modules to application library in CICS DFHRPL DD concatenation  
- Bind updated DB2 DBRMs/Packages to DB2 plan
- Do CICS refresh (New Copy) to pick up updated load modules  
- In the event of failure, rollback to old load modules and DBRMs

Implement the CICS-COBOL-DB2 application deployment steps with Ansible playbooks as mentioned below:

1. Define the Ansible environment configuration in “[ansible.cfg](https://github.ibm.com/Anuprakash-Moothedath/Mainframe-Applications-deployments-using-Ansible/blob/master/Ansible-IBM-Z-App-Deploy/ansible.cfg)” file and use “[inventory](https://github.ibm.com/Anuprakash-Moothedath/Mainframe-Applications-deployments-using-Ansible/blob/master/Ansible-IBM-Z-App-Deploy/inventory)“ file to define z/OS managed node and to provide the Python interpreter path in z/OS USS. Also define environment variables required for Ansible to work with z/OS in a variable file “[./group_vars/all.yml](https://github.ibm.com/Anuprakash-Moothedath/Mainframe-Applications-deployments-using-Ansible/blob/master/Ansible-IBM-Z-App-Deploy/group_vars/all.yml)”  

2. Define the variables that can be used in the deployment playbook in a variable file, “[./group_vars/deploy_vars.yml](https://github.ibm.com/Anuprakash-Moothedath/Mainframe-Applications-deployments-using-Ansible/blob/master/Ansible-IBM-Z-App-Deploy/group_vars/deploy_vars.yml)”. This is for the  ease of maintaining the playbook. Following is an excerpt of the variable file:
```YAML
# Is the application Batch, DB2, CICS or both.  Set true or false as per your application.
cics: true
db2: true

# Load modules - Source, Target and backup PDS and modified modules
load_src: #<Load Module source PDS>
load_tgt: #<Load Module target PDS>
load_bkp: #<Load Module backup PDS>
load_mems: #<List of modified Load modules to deploy, Ex:['LOAD01','LOAD02','LOAD03']>

# DBRMs -  Source, Target and backup PDS and modified DBRMs
dbrm_src: #<DBRM source PDS>
dbrm_tgt: #<DBRM target PDS>
dbrm_bkp: #<DBRM backup PDS>
dbrms: #<List of modifies DBRMs to BIND, Ex: ['DBRM01','DBRM02','DBRM03']>

# DB2 parameters if DB2 is set to true
db2_subsys: #<DB2 Subsystem name>
db2_package: #<DB2 Package name>
db2_plan: #<DB2 Plan name>

# CICS programs if CICS is set to true
cics_pgms: #<List of modified CICS programs to deploy, Ex:['LOAD01','LOAD02','LOAD03']>
```

3. Create an Ansible playbook - “[deploy_app.yml](https://github.ibm.com/Anuprakash-Moothedath/Mainframe-Applications-deployments-using-Ansible/blob/master/Ansible-IBM-Z-App-Deploy/deploy_app.yml)” that when executed can deploy the application. The variables “db2” and “cics” defined in “[./group_vars/deploy_vars.yml](https://github.ibm.com/Anuprakash-Moothedath/Mainframe-Applications-deployments-using-Ansible/blob/master/Ansible-IBM-Z-App-Deploy/group_vars/deploy_vars.yml)”, can help in deciding whether DB2 bind or CICS refresh or both DB2 bind and CICS refresh are required to complete application deployment. This helps to use the same playbook with multiple application sub-types CICS-COBOL, COBOL-DB2 or just COBOL. Following is an excerpt of the Ansible playbook:
```YAML
- name: Deploy Z Application
  hosts: all

  gather_facts: no

  collections:
    - ibm.ibm_zos_core
  
  vars_files: "./group_vars/deploy_vars.yml"

  environment: "{{ environment_vars }}"  
  
  tasks:  
    - name: Backup Load modules in target PDS
      include_tasks: "./tasks/bkp_load.yml"

    - name: Copy Load modules to target PDS
      include_tasks: "./tasks/copy_load.yml"

    - name: Backup DBRMs in target PDS
      include_tasks: "./tasks/bkp_dbrm.yml"
      when: (db2 == true)

    - name: Copy DBRMs to target PDS
      include_tasks: "./tasks/copy_dbrm.yml"    
      when: (db2 == true)

    - name: Bind DBRMs to DB2 Plan
      include_tasks: "./tasks/db2_bind.yml"    
      when: (db2 == true)

    - name: CICS Refresh
      include_tasks: "./tasks/cics_refresh.yml"    
      when: (cics == true)    
```

4.	One way to backup and copy the load modules and DBRMs from source libraries to backup libraries and target libraries is using IEBCOPY utility. Use Ansible template module and Jinja2 templating to dynamically generate the JCL required to backup/copy load modules.  Create Jinja2 template - “[./files/LOADBKUP.j2](https://github.ibm.com/Anuprakash-Moothedath/Mainframe-Applications-deployments-using-Ansible/blob/master/Ansible-IBM-Z-App-Deploy/files/LOADBKUP.j2)” for JCL to backup load module. All the variables inside double curly braces ```{{}}``` in the Jinja2 template can substitute the values from the variable file “[./group_vars/deploy_vars.yml](https://github.ibm.com/Anuprakash-Moothedath/Mainframe-Applications-deployments-using-Ansible/blob/master/Ansible-IBM-Z-App-Deploy/group_vars/deploy_vars.yml)”. The substitution is performed by Ansible template module during the playbook execution. The ```for``` loop in the SYSIN can replicate the select statement for each of the load members to be backed up during the execution. Following is an excerpt of the Jinja2 template for backing up load modules:
```JCL
//COPYLOAD EXEC PGM=IEBCOPY,REGION=4M          
//*MAIN SYSTEM=JLOCAL,LINES=99                 
//SYSPRINT DD SYSOUT=*                         
//SYSUT1 DD DSN={{ load_tgt }},     
//       DISP=SHR                              
//SYSUT2 DD DSN={{ load_bkp }},      
//       DISP=SHR                              
//SYSUT3   DD UNIT=VIO,SPACE=(CYL,(10))        
//SYSUT4   DD UNIT=VIO,SPACE=(CYL,(10))        
//SYSIN    DD *                                
  COPYMOD INDD=SYSUT1,OUTDD=SYSUT2,MAXBLK=32760
{% for load_mem in load_mems %}
  SELECT M=(({{ load_mem }},,R))    
{% endfor %}                   
/* 
```

5. Create a task to back-up load modules - “[./tasks/bkp_load.yml](https://github.ibm.com/Anuprakash-Moothedath/Mainframe-Applications-deployments-using-Ansible/blob/master/Ansible-IBM-Z-App-Deploy/tasks/bkp_load.yml)”. This task can dynamically generate JCL to backup load modules in z/OS using the Jinja2 template with Ansible template module and then subsequently execute the generated JCL. Following is an excerpt of load modules backup task.
```YAML
- name: Job template render for LOAD module backup
  template:
    src: './files/LOADBKUP.j2'
    dest: '/tmp/LOADBKUP.bin'
  register: result

- name: LOAD module backup - Template Response
  debug:
    msg: "{{ result }}"
       
- name: Convert LOAD backup JCL encoding from ISO8859-1 to IBM-1047
  zos_encode:
    src: '/tmp/LOADBKUP.bin'
    dest: '/tmp/LOADBKUP.jcl'
    to_encoding: IBM-1047
    from_encoding: ISO8859-1
  register: result

- name: LOAD backup JCL Encoding conversion response
  debug:
    msg: "{{ result }}"   

- name: Submit the job from USS
  zos_job_submit:
    src: '/tmp/LOADBKUP.jcl'
    location: USS
    wait: true
  register: result

- name: Job Response
  debug:
    msg: "{{ result.jobs[0].ret_code }}"
```

6. Develop JCL templates and tasks to copy the load modules to target libraries – “[./tasks/copy_load.yml](https://github.ibm.com/Anuprakash-Moothedath/Mainframe-Applications-deployments-using-Ansible/blob/master/Ansible-IBM-Z-App-Deploy/tasks/copy_load.yml)”, backing up DBRMs – “[./tasks/bkp_dbrm.yml](https://github.ibm.com/Anuprakash-Moothedath/Mainframe-Applications-deployments-using-Ansible/blob/master/Ansible-IBM-Z-App-Deploy/tasks/bkp_dbrm.yml)” and copying DBRMs to target libraries – “[./tasks/copy_dbrm.yml](https://github.ibm.com/Anuprakash-Moothedath/Mainframe-Applications-deployments-using-Ansible/blob/master/Ansible-IBM-Z-App-Deploy/tasks/copy_dbrm.yml)”.  These tasks are very similar to step 4 and step 5.

7. Use Ansible template module and Jinja2 templating to generate DB2 bind JCL. Create a Jinja2 template - “[./files/DB2BIND.j2](https://github.ibm.com/Anuprakash-Moothedath/Mainframe-Applications-deployments-using-Ansible/blob/master/Ansible-IBM-Z-App-Deploy/files/DB2BIND.j2)” for DB2 bind JCL. All variables inside double curly braces ```{{}}``` in Jinja2 template can substitute with the values from the variable file “[./group_vars/deploy_vars.yml](https://github.ibm.com/Anuprakash-Moothedath/Mainframe-Applications-deployments-using-Ansible/blob/master/Ansible-IBM-Z-App-Deploy/group_vars/deploy_vars.yml)” during execution. This substitution is done by Ansible template module. The ```for``` loop in the SYSIN can replicate select statement for each of the load members that needs back-up. Following is an excerpt of Jinja2 template for DB2 Bind JCL:
```JCL
//BIND    EXEC PGM=IKJEFT01,DYNAMNBR=20
//STEPLIB  DD  DSN=DSNC10.SDSNLOAD,DISP=SHR
//DBRMLIB  DD  DSN={{ dbrm_tgt }},DISP=SHR
//SYSPRINT DD  SYSOUT=*
//SYSTSPRT DD  SYSOUT=*
//SYSUDUMP DD  SYSOUT=*
//SYSIN  DD *
/*
//SYSTSIN DD *
DSN SYSTEM(DBCG)
{% for dbrm in dbrms %}
BIND PACKAGE ({{ db2_package }})                          -
     ISO(CS)                                              -
     CURRENTDATA(NO)                                      -
     MEMBER({{ dbrm }})                                   -
     DEGREE(1)                                            -
     DYNAMICRULES(BIND)                                   -
     ACTION (REPLACE)                                     -
     EXPLAIN(NO)                                          -
     OWNER(ADCDMST)                                       -
     QUALIFIER(GENAPP1)                                   -
     ENABLE(BATCH,CICS)                                   -
     REL(DEALLOCATE)                                      -
     VALIDATE(BIND)
{% endfor %}

BIND PLAN (GENAONE)                                       -
     PKLIST(NULLID.*, *.{{ db2_package }}.*)              -
     CURRENTDATA(NO)                                      -
     ISO(CS)                                              -
     ACTION (REP)                                         -
     OWNER(ADCDMST)                                       -
     QUALIFIER(GENAPP1)                                   -
     REL(DEALLOCATE)                                      -
     ACQUIRE(USE)                                         -
     RETAIN                                               -
     NOREOPT(VARS)                                        -
     VALIDATE(BIND)

RUN PROGRAM(DSNTIAD) PLAN({{ db2_plan }}) -
    LIB('DSNC10.{{ db2_subsys }}.RUNLIB.LOAD')
END
/*
```

8. Create an Ansible task – “[./tasks/db2_bind.yml](https://github.ibm.com/Anuprakash-Moothedath/Mainframe-Applications-deployments-using-Ansible/blob/master/Ansible-IBM-Z-App-Deploy/tasks/db2_bind.yml)” to generate DB2 BIND JCL and copy JCL to z/OS USS path and execute using the Ansible playbook. This is very similar to what is mentioned in step 5 for load modules backup. 

9. Do the CICS refresh (new copy) from Ansible playbook by executing ```CEMT``` transaction in the target CICS region. To achieve this using Ansible, use ```zos_operator module``` to issue ```CEMT``` as a console command in the playbook task, “[./tasks/cics_refresh.yml](https://github.ibm.com/Anuprakash-Moothedath/Mainframe-Applications-deployments-using-Ansible/blob/master/Ansible-IBM-Z-App-Deploy/tasks/cics_refresh.yml)”. The variable ```cics_pgms``` can resolve from variables file – “[./group_vars/deploy_vars.yml](https://github.ibm.com/Anuprakash-Moothedath/Mainframe-Applications-deployments-using-Ansible/blob/master/Ansible-IBM-Z-App-Deploy/group_vars/deploy_vars.yml)”. The CICS new copy can be done for all the programs listed in ```cics_pgms``` in the variable file. Following is an excerpt of the CICS refresh task.
```YAML
- name: New Copy CICS Program
  zos_operator:
    cmd: 'F CICSTS55,CEMT S PROG({{ item }}) NEW'
  loop: "{{ cics_pgms }}"
  register: result

- name: Response for New Copy
  debug:
    msg: "{{ result }}"
```

10. Do the rollback of application to reverse the changes in target libraries if required. Create an Ansible task, “[rollback_app.yml](https://github.ibm.com/Anuprakash-Moothedath/Mainframe-Applications-deployments-using-Ansible/blob/master/Ansible-IBM-Z-App-Deploy/rollback_app.yml)” to copy back the previous version of the load modules and DBRMs from the backup libraries to the target environment libraries. The DBRMs can be bind to the DB2 plan and the CICS refresh (new copy) can be done to the CICS programs based on the parameters ```db2``` and ```cics``` defined in variables file - “[./group_vars/deploy_vars.yml](https://github.ibm.com/Anuprakash-Moothedath/Mainframe-Applications-deployments-using-Ansible/blob/master/Ansible-IBM-Z-App-Deploy/group_vars/deploy_vars.yml)”. 

### DevOps integration: Versioning and integrating Ansible playbooks in CI-CD pipeline
When we think of version controlling and integrating Ansible playbooks in DevOps pipeline we need an SCM tool and a CI-CD pipeline orchestrator tool. Git based repositories like Github, Gitlab, and Bitbucket are some of the most widely used version control systems. Jenkins is one of the most popular CI-CD pipeline orchestrators, which forms DevOps' backbone. We can see these tools in most businesses that adopted DevOps.  Now, we can go through various steps to integrate Ansible playbooks into the CI-CD pipeline. 

We can use GitHub as SCM to version control our Ansible playbooks, as it is a popular SCM that can easily integrate with Jenkins orchestrator and work seamlessly with the mainframe. However, you can use any SCM that can integrate with your existing CI-CD pipeline orchestrator and Ansible.   
In Jenkins, Ansible plugin is available using which we can execute Ansible playbooks. If you don’t want to use Ansible plugin, you can also execute the Ansible playbook from the shell or windows batch command under build step in Jenkins. 

We use Ansible plugin in Jenkins to execute Ansible playbooks, to install Ansible plugin go to Manage Jenkins ➡ Manage Plugins ➡ Available Plugins ➡ Filter Ansible and Install.

If you want to install Ansible plugin via cli please visit - [Ansible | Jenkins plugin](https://plugins.jenkins.io/ansible/).  

![01_Ansible_Plugin.PNG](https://github.ibm.com/Anuprakash-Moothedath/Mainframe-Applications-deployments-using-Ansible/blob/master/Images/01_Ansible_Plugin.PNG)

Do the following configurations in Jenkins to integrate with Ansible:
 
1. Define the Ansible installation path in Jenkins from Manage Jenkins ➡ Global Tool Configuration ➡ Ansible ➡ Ansible installations ➡ Add Ansible and provide name and path to the Ansible executables.

![02_Ansible_Config.PNG](https://github.ibm.com/Anuprakash-Moothedath/Mainframe-Applications-deployments-using-Ansible/blob/master/Images/02_Ansible_Config.PNG)

2. Install Ansible z/OS core collections under Jenkins user by creating a Jenkins job and by executing the installation command from ```Execute shell``` option under ```Add build``` as show in the below figure. Multiple shell commands can be executed as needed.  This is a one-time activity.

![03_Install_zos_core_in_Jenkins.PNG](https://github.ibm.com/Anuprakash-Moothedath/Mainframe-Applications-deployments-using-Ansible/blob/master/Images/03_Install_zos_core_in_Jenkins.PNG)

3. Create a Jenkins job and add Git as source code management and provide the URL of our Git repository containing Ansible playbooks. This is to pull Ansible playbook repository from Git.

![04_SCM_pull_Ansible_Playbook_in_Jenkins.PNG](https://github.ibm.com/Anuprakash-Moothedath/Mainframe-Applications-deployments-using-Ansible/blob/master/Images/04_SCM_pull_Ansible_Playbook_in_Jenkins.PNG)

4. Add a build step to “invoke Ansible playbook”. Choose the Ansible installation path given in the Global Tool configuration. Provide the playbook path in Jenkins workspace. Provide all other parameters needed under this build step. 

![05_Execute_Ansible_Playbook_in_Jenkins.PNG](https://github.ibm.com/Anuprakash-Moothedath/Mainframe-Applications-deployments-using-Ansible/blob/master/Images/05_Execute_Ansible_Playbook_in_Jenkins.PNG)

5. Execute the playbook from Jenkins by running the Jenkins job defined. We can trigger this Jenkins job in many ways like webhook triggers, calling Jenkins rest API for job build, using scheduler, manual trigger etc.

To understand more about our playbooks, tasks, templates and files, you can visit the git repo [Ansible-IBM-Z-App-Deploy](https://github.com/anuprakashm/Mainframe-Applications-deployments-using-Ansible/tree/master/Ansible-IBM-Z-App-Deploy)

## Summary
Ansible is a simple yet powerful automation tool, which makes it a very popular open-source automation tool. With Ansible coming to IBM Z and z/OS, you can extend all the goodies of Ansible to z/OS platform including platform agnostic integrated DevOps.

## Related Links
You can read more on Ansible here - [Ansible Documentation](https://docs.ansible.com/).
You can read on Ansible collections for z/OS here - [Ansible Galaxy](https://galaxy.ansible.com/search?deprecated=false&keywords=zos&order_by=-relevance&page=1).