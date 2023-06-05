## This ansible playbook installs [`Jenkins`](https://www.jenkins.io/doc/) on specified host ##

# Prerequisites
* Run the ansible playbook on `Debian` or `Ubuntu`. [Used was VM with Jammy Ubuntu](https://github.com/Alliedium/awesome-proxmox). Use the [script](https://github.com/Alliedium/awesome-proxmox/blob/main/vm-cloud-init-shell/.env.example) to create VM on `Proxmox`.  

* Use `$HOME/awesome-jenkins/inventory/localhost/hosts.yaml` if you are installing the `Jenkins` on the same host where `Ansible` is running.  Use `$HOME/awesome-jenkins/inventory/example/hosts.yaml` if you are installing the `Jenkins` on the remote host.
  
  In our examples, we use `$HOME/awesome-jenkins/inventory/localhost/hosts.yaml` file.

* Install Ansible: [Follow the second step](https://github.com/Alliedium/awesome-ansible#setting-up-config-machine)

* [Install `molecule`](https://molecule.readthedocs.io/installation/) on `Ubuntu` Linux. Molecule project is designed to aid in the development and testing of Ansible roles.
  
   ```
   apt update
   apt install pip
   python3 -m pip install molecule ansible-core
   pip3 install 'molecule-plugins[docker]'
   ```

## Playbook variables used in Jenkins server installation:

1. The HTTP port for Jenkins' web interface:

   ```
   jenkins_http_port: 8085
   ```

2. Admin account credentials which will be created the first time Jenkins is installed:

   ```
   jenkins_admin_username: admin
   jenkins_admin_password: admin
   ```

3. Java version:
   
   ```   
   java_packages: 
     - openjdk-17-jdk
   ```

4. Install global tools. Maven versions:
    
   ```
   jenkins_maven_installations:
     - 3.8.4
     - 3.9.0
   ```

5. [List of plugins that will be installed](ListofJenkinsPluginsToBeInstalled.md)

## Instructions to install Jenkins with ansible playbook

### 1. Clone repo:

  ```
  git clone https://github.com/Alliedium/awesome-jenkins.git $HOME/awesome-jenkins
  ```
### 2. Installing 'Jenkins' on remote host

* Copy `$HOME/awesome-jenkins/inventory/example` to `$HOME/awesome-jenkins/inventory/my-jenkins` folder.
  
  ```
  cp -r $HOME/awesome-jenkins/inventory/example $HOME/awesome-jenkins/inventory/my-jenkins
  ```

* Change the variables in the files `$HOME/awesome-jenkins/inventory/my-jenkins/hosts.yml` as you need

![](./images/hosts.png)

* Installing `Jenkins` on localhost does not require any changes to `$HOME/awesome-jenkins/inventory/localhost/hosts.yml` file.

### 3. Install ansible roles for [Java](https://github.com/geerlingguy/ansible-role-java/), [Git](https://github.com/geerlingguy/ansible-role-git/), and [Jenkins](https://github.com/geerlingguy/ansible-role-jenkins) using commands:
   
   ```
   ansible-galaxy install -r $HOME/awesome-jenkins/requirements.yml
   ```

### 4. Run ansible playbook 

  This playbook contains multiple tasks that install `git`, `java`, `Jenkins`, as well as plugins, tools and pipelines in `Jenkins`. Using `Ansible` tags you can run a part of tasks. In our playbook we use 7 tags: `always`, `step1`, `step2`, `step3`, `step4`, `step5` and `step6`. Use `-t <tag_name>` flag to specify desired tag. They form a hierarchy of tags from `always` to `step6`. In this hierarchy, each subsequent tag includes both the tasks marked by this tag as well as tasks relating to all preceding tags, e.g. if you run playbook with `step3` tag, tasks tagged with `always`, `step1`, `step2` and `step3` will be run.

   1. Before running tasks, check the list of tasks that will be executed using `--list-tasks` flag
   
   ```
   ansible-playbook $HOME/awesome-jenkins/playbooks/create-job.yml -i $HOME/awesome-jenkins/inventory/localhost --list-tasks
   ```

   You will receive a list of all tasks. Using `-t step2` when getting a list of tasks.

   ```
   ansible-playbook $HOME/awesome-jenkins/playbooks/create-job.yml -i $HOME/awesome-jenkins/inventory/localhost -t step2 --list-tasks
   ```

   You will receive a list of tasks, tagged `always`, `step1` and `step2`.


   2. Run all the available tasks from `playbook.yml` playbook. 
   
   ```
   ansible-playbook $HOME/awesome-jenkins/playbooks/create-job.yml -i $HOME/awesome-jenkins/inventory/localhost
   ```
   3. Run without installing any plugins in `Jenkins`:
   
   ```
   ansible-playbook $HOME/awesome-jenkins/playbooks/create-job.yml -i $HOME/awesome-jenkins/inventory/localhost -t step1
   ```

   4. Run with installing [plugins](ListofJenkinsPluginsToBeInstalled.md) in `Jenkins`:
   
   ```
   ansible-playbook $HOME/awesome-jenkins/playbooks/create-job.yml -i $HOME/awesome-jenkins/inventory/localhost -t step2
   ```

   5. Use `step3` tag - install `python-jenkins`
   
   ```
   ansible-playbook $HOME/awesome-jenkins/playbooks/create-job.yml -i $HOME/awesome-jenkins/inventory/localhost -t step3
   ```

   6. `step4` - Add  `maven` tool
   
   ```
   ansible-playbook $HOME/awesome-jenkins/playbooks/create-job.yml -i $HOME/awesome-jenkins/inventory/localhost -t step4
   ```

   7. `step5` - Create and launch  `Jenkins pipeline job`
   
   ```
   ansible-playbook $HOME/awesome-jenkins/playbooks/create-job.yml -i $HOME/awesome-jenkins/inventory/localhost -t step5
   ```
   
   8. `step6 - Create and launch  `Jenkins multibranch pipeline job`
   
   ```
   ansible-playbook $HOME/awesome-jenkins/playbooks/create-job.yml -i $HOME/awesome-jenkins/inventory/localhost -t step6
   ```

### 5. Check `Jenkins`

1. Go to the host specified in the `$HOME/awesome-jenkins/inventory/localhost/hosts.yml` file, open browser and check that Jenkins is available at http://localhost:8085/.
2. Login to Jenkins using the credentials.
3. You will see Jenkins dashboard. Open job. ![jenkins_dashboard.png](./images/01jenkins_dashboard.png) 
4. The main branch will be run for the single pipeline job ![single_pipeline.png](./images/02jenkins_pipeline.png)
5. Pull requests will be run for the multibranch pipeline job.![multibranch_pipeline.png](./images/03jenkins_mpipeline.png)

### 5. Ansible playbook local testing with [molecule](https://molecule.readthedocs.io/)

The `molecule` configuration files are located in the `$HOME/awesome-jenkins/molecule/default` folder.

`molecule.yml` - this is the core file for Molecule. Used to define your testing steps, scenarios, dependencies, and other configuration options.

`converge.yml` - this is the playbook that Molecule will run to provision the targets for testing.

`verify.yml` - this is the playbook that is used to validate that the already converged instance state matches the desired state. 

Before running the `molecule` command, go to `awesome-jenkins` project

```
cd $HOME/awesome-jenkins
```

* Run Ansible playbook test after which all previously created resources are deleted.
  
```
molecule test
```

The `test` command will run the entire scenario; creating, converging, verifying.

* Ansible playbook execution or role in target infrastructure, without testing. In this case, molecule will run the Ansible playbook in docker
  
```
molecule converge
```

* Run Ansible playbook test after the infrastructure has been converged using the "molecule converge" command. All previously created resources are not deleted
  
```
molecule verify
```

* Navigate to the target infrastructure - the docker container with the debug or check target

```
molecule login
```

* Reset molecule temporary folders.

```
molecule reset
```

* Finally, to clean up, we can run

```
molecule destroy
```

This removes the containers that we deployed and provisioned with create or converge. Putting us into a great place to start again.

### 6. Ansible playbook remote testing with github Actions

The `$HOME/awesome-jenkins/.github/workflows/ci.yml` file describes the steps for `github` Actions testing.

After creating or updating a pull request, tests are launched on the `github` server and the results can be viewed here

![github_actions](./images/github_actions.png)

![github_actions_1](./images/github_actions_1.png)

## Jenkins and `github` integration

### 1. Make sure the `GitHub Checks` plugin and its dependent plugins are installed on `Jenkins`

![plugins](./images/plugins.png)

### 2. Set Resource Root URL

![resource_root_url](./images/resource_root_url.png)

### 3. Creating your organization in `github`
  
  ![creating_org_1](./images/creating_org_1.png)

  ![creating_org_2](./images/creating_org_2.png)

### 4. Creating `github apps`

![github_app](./images/github_app.png)

### 5. Generate and download SSH key

![](./images/ssh_key.png)

### 6. Install your app for repositories

![install_app](./images/install_app.png)

### 7. Convert your generated key

```
openssl pkcs8 -topk8 -inform PEM -outform PEM -in key-in-your-downloads-folder.pem -out converted-github-app.pem -nocrypt
```

`key-in-your-downloads-folder.pem` - your generated SSH key

`converted-github-app.pem` - converted key

### 8. Fork your repo for testing purposes on `github`

  ![fork](./images/fork.png)

### 9. Create `multibranch pipeline` in `Jenkins`

![mpipeline](./images/mpipeline.png)

![mp_config](./images/mp_config_2.png)

### 10. On `github` create new branch and pull request

After creating new pull request on `Jenkins` scan repository

![scan_repository](./images/scan_repository.png)

### 11. Run you build

![run_pr](./images/run_pr.png)

### 12. See build result on `github`

![github_checks](./images/github_checks.png)

## Project:
   As the example we used the following [project](https://github.com/Alliedium/springboot-api-rest-example)

## Job configuration:
   Job configuration is set in the `templates/job-config.xml.j2` and `templates/job-config-2.xml.j2`.
