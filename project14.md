## Setting the stage
- Setup Jenkins Server RHEL
- Clone Ansible Repo
- Install epel-release
- Install remi repo
- Install JAVA
- Open .bash_profile
- Insert Java Path
  ```
  export JAVA_HOME=$(dirname $(dirname $(readlink $(readlink $(which javac))))) 
  export PATH=$PATH:$JAVA_HOME/bin 
  export CLASSPATH=.:$JAVA_HOME/jre/lib:$JAVA_HOME/lib:$JAVA_HOME/lib/tools.jar
  ```
- Reload bash profile `source ~/.bash_profile`
- Install Jenkins `sudo yum install jenkins -y`
- Start Jenkins `sudo systemctl start jenkins` `sudo systemctl enable jenkins` `sudo systemctl daemon-reload`
- Use browser to visit `jenkins-server-public-ip:8080`
- Install & Open Blue Ocean plugin
- Create New Pipeline
- Connect to Github Repo & add access token
- Create Jenkinsfile manually inside `deploy` directory in `ansible-config-mgt` project
- Add code snippet to test
  ```
  pipeline {
    agent any

    stages {
      stage('Build') {
        steps {
          script {
            sh 'echo "Building Stage"'
          }
        }
      }
    }
  }
  ```
- Return to Jenkins Admin Dashboard -> Select project -> Configure
- Under Build Configuration, enter path to `Jenkinsfile` in Script Path field: `deploy/Jenkinsfile`. Save.
- Return to pipeline project and click `Build Now`
- Create new git branch `feature/jenkinspipeline-stages`
- Add stage to Jenkinsfile:
  ```
  pipeline {
      agent any

      stages {
          stage('Build') {
              steps {
                  script {
                      sh 'echo "Building Stage"'
                  }
              }
          }
          stage('Test') {
              steps {
                  script {
                      sh 'echo "Testing Stage"'
                  }
              }
          }
      }
  }
  ``` 
- Scan repo to have new stage reflected in Jenkins dashboard
- Ensure build is successful in Jenkins
- Do the following:
  ```
  1. Create a pull request to merge the latest code into the main branch
  2. After merging the PR, go back into your terminal and switch into the main branch.
  3. Pull the latest change.
  4. Create a new branch, add more stages into the Jenkins file to simulate below phases. (Just add an echo command like we have in build and test stages)
     1. Package 
     2. Deploy 
     3. Clean up
  5. Verify in Blue Ocean that all the stages are working, then merge your feature branch to the main branch
  6. Eventually, your main branch should have a successful pipeline like this in blue ocean
  ```

## Running Ansible From Jenkins
- Install Ansible on Jenkins server
- Install Ansible plugin on Jenkins Dashboard
- Configure Ansible in `Global Tools Configuration` under `Manage Jenkins`
- Insert path to ansible: `/usr/bin`. Save
- Spin up 2 Servers for Nginx (RHEL 8) and Database (Ubuntu)
- Place `ansible.cfg` file into `deploy` directory. Insert the following configuration:
  ```
  [defaults]
  timeout = 160
  callback_whitelist = profile_tasks
  log_path=~/ansible.log
  host_key_checking = False
  gathering = smart
  ansible_python_interpreter=/usr/bin/python3
  allow_world_readable_tmpfiles=true

  [ssh_connection]
  ssh_args = -o ControlMaster=auto -o ControlPersist=30m -o ControlPath=/tmp/ansible-ssh-%h-%p-%r -o ServerAliveInterval=60 -o ServerAliveCountMax=60 -o ForwardAgent=yes
  ```
- NOTE: Don't define `roles_path` in config file. Rather, define it using Jenkins & environment variables
- Define environment variable for ansible configuration file in Jenkinsfile:
  ```
  pipeline {
    agent any
    environment {
      ANSIBLE_CONFIG="${WORKSPACE}/deploy/ansible.cfg"
    }
    stages {...}
  }
  ```
- Proceed to add stages and steps to Jenkinsfile to execute ansible playbook
  ```
  pipeline {
    agent any
    environment {
      ANSIBLE_CONFIG="${WORKSPACE}/deploy/ansible.cfg"
    }
    stages {
      stage ('Initial Cleanup') {
        steps {
          dir("${WORKSPACE}"){
            deleteDir()
          }
        }
      }
      stage ('Checkout SCM') {
        steps {
          git branch: 'main', url: 'https://github.com/mrdankuta/ansible-config-mgt.git'
        }
      }
      stage ('Prepare Ansible for Execution') {
        steps {
          sh 'echo ${WORKSPACE}'
          sh 'sed -i "3 a roles_path=${WORKSPACE}/roles" ${WORKSPACE}/deploy/ansible.cfg'
        }
      }
      stage ('Run Ansible Playbook') {
        steps {
          ansiblePlaybook become: true, credentialsId: 'ansible_target_1', disableHostKeyChecking: true, installation: 'jensible', inventory: 'inventory/dev', playbook: 'playbooks/site.yml'
        }
      }
      stage ('Cleanup After Build') {
        steps {
          cleanWs(cleanWhenAborted: true, cleanWhenFailure: true, cleanWhenNotBuilt: true, cleanWhenUnstable: true, deleteDirs: true)
        }
      }
    }
  }
  ```
- Setup Credentials for Ansible on Jenkins Dashboard by going to `Manage Jenkins` -> `Credentials` -> `Global` -> `Add New`. Choose `SSH Username with Private Key` option. Enter SSH Username and private key.
- Generate code for the `Run Ansible Playbook` by going to `ansible-config-mgt` project -> `Pipeline Syntax`.
- - Choose `ansiblePlaybook: Invoke an ansible playbook`
- - Choose ansible plugin config in `Ansible tool`
- - Enter Path to Playbooks entry file `playbooks/site.yml`
- - Enter path to Inventory file `inventory/dev`
- - Select SSH Connection Credentials 
- - Select `Use become`
- - Select `Disable the host SSH key check`
- - Click `Generate Pipeline Script` and copy output and insert as step for the stage in Jenkinsfile
- Update inventory with correct IP address
- Commit and push to `feature/jenkinspipeline-stages` branch
- Scan repository in Jenkins dashboard to test run
- If pipeline build is successful, proceed to parameterize the setup to allow being able to specify what inventory environment to run the pipeline. To do this, goto `Jenkinsfile` and add a global parameter in pipeline:
  ```
  parameters {
    string(name: 'inventory', defaultValue: 'dev',  description: 'This is the inventory file for the environment to deploy configuration')
  }
  ```
- In `Run Ansible Playbook` stage, replace `inventory/dev` with `inventory/${inventory}`. Commit & push to remote repository. Scan repository from Jenkins and ensure it is successfull.

## CI/CD Pipeline for Todo Application
- Fork the Todo app in Github `https://github.com/darey-devops/php-todo.git` to your account.
- Clone app to your computer
  ```sh
  git clone https://github.com/mrdankuta/php-todo.git
  ```
- Install PHP and necessary dependencies on Jenkins-Ansible server
  ```sh
  sudo su -
  yum module reset php -y
  yum module enable php:remi-7.4 -y
  yum install -y php php-common php-mbstring php-opcache php-intl php-xml php-gd php-curl php-mysqlnd php-fpm php-json
  systemctl start php-fpm
  systemctl enable php-fpm
  ```
- Install Composer
  ```sh
  curl -sS https://getcomposer.org/installer | php
  mv composer.phar /usr/bin/composer
  composer --version
  ```
- Return to Jenkins Dashboard. Install `Plot` plugin and `Artifactory` plugin
- Launch a new server instance for `Artifactory`. Should be a minimum of `4GB Memory` and `2 vCPUs`
- Install Artifactory by using an Ansible Role. Use the following steps and configuration code to setup the role:
- - Create a role directory structure and name it `artifactory` manually or use the `ansible-galaxy init artifactory` command to create the role directory structure automatically.
- - In `defaults/main.yml` insert:
  ```
  code here...
  ```
- - In `handlers/main.yml` insert:
  ```
  code here...
  ```
- - In `tasks/main.yml` insert:
  ```
  code here...
  ```
- - In `templates/bash-profile.j2` insert:
  ```
  code here...
  ```
- - In `defaults/main.yml` insert:
  ```
  code here...
  ```
- Update `ci` inventory file with internal IP Address of artifactory server. Name the host group `artifactory`
- Create assignment for artifactory in `static-assignments/artifactory.yml`:
  ```yml
  ---
  - hosts: artifactory
    become: true
    roles:
      - artifactory
  ```
- Add assignment to entry playbook `playbooks/site.yml`
  ```yml
  - name: artifactory assignment
    ansible.builtin.import_playbook: ../static-assignments/artifactory.yml
  ```
- Return to project on Jenkins dashboard. Navigate to `Build with Parameters`
- Enter `ci` inventory file name and click `Build`
- After successful build, visit artifactory dashboard in browser with `<public-ip-address>:8081`. Default username is `admin`. Default password is `password`. Ensure firewall is open to ports `8081` and `8082`
- Create new password with welcome prompt. Skip through onboarding prompts.
- Create new local repository. Enter a name without spaces or special characters.
- Return to Jenkins dashboard. Navigate to `Manage Jenkins -> Configure System `. Scroll to JFrog Artifactory section. Click `Add JFrog Instance`
- Enter an `Instance ID`. `artifactory-server` will do.
- Enter artifactory url `<public-ip-address>:8081` in `JFrog Platform URL` field
- Enter artifactory username and password in `Default Deployer Credentials`
- Click `Test Connect`. If successful, save.


## Integrate Artifactory Repository with Jenkins
- Create `Jenkinsfile` inside `php-todo` directory and insert the following code:
  ```
  pipeline {
      agent any

    stages {

      stage("Initial cleanup") {
            steps {
              dir("${WORKSPACE}") {
                deleteDir()
              }
            }
          }

      stage('Checkout SCM') {
        steps {
              git branch: 'main', url: 'https://github.com/mrdankuta/php-todo.git'
        }
      }

      stage('Prepare Dependencies') {
        steps {
              sh 'mv .env.sample .env'
              sh 'composer install'
              sh 'php artisan migrate'
              sh 'php artisan db:seed'
              sh 'php artisan key:generate'
        }
      }
    }
  }
  ```
- Create database and user on database server using the `mysql` ansible role. Add the following to `roles/mysql/defaults/main.yml`:
  ```
  mysql_databases:
    - name: homestead
      collation: utf8_general_ci
      encoding: utf8
      replicate: 1
  ```
  ```
  mysql_users:
    - name: homestead
      host: <jenkins-server-private-ip-address>
      password: sePret^i
      priv: '*.*:ALL,GRANT'
  ```
  This is a client-server architecture that allows Jenkins use the database server as a client. 
  - Hence, MySQL client needs to be installed on Jenkins Server `sudo yum install mysql -y`
  - Log in to database server and set the `bind-address` to `0.0.0.0` to allow remote connection.
    ```
    sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf
    ```
  - Restart MySQL server `sudo systemctl restart mysql`
- In `php-todo` directory, update database connection details in `.env.sample`:
  ```
  DB_HOST=<db-server-private-ip>
  DB_DATABASE=homestead
  DB_USERNAME=homestead
  DB_PASSWORD=sePret^i
  DB_CONNECTION=mysql
  DB_PORT=3306
  ```
- Push code to remote repo and run ansible from Jenkins Dashboard against the `dev` environment to execute the creation of the database and user above. You can log into database server to be sure database and user were created
  ```
  ssh -i private-key user@db-server-ip
  sudo mysql
  show databases;
  select user, host from mysql.user;
  ```
- Add Unit tests stage/step in `php-todo/Jenkinsfile` pipeline
  ```
  stage('Execute Unit Tests') {
    steps {
      sh './vendor/bin/phpunit'
    }
  }
  ```
- Add code analysis stage/step in `php-todo/Jenkinsfile` pipeline
  ```
  stage('Code Analysis') {
    steps {
      sh 'phploc app/ --log-csv build/logs/phploc.csv'
    }
  }
  ```
- Add stage/step for plotting report in `php-todo/Jenkinsfile` pipeline. _This plugin provides generic plotting (or graphing) capabilities in Jenkins. It will plot one or more single values variations across builds in one or more plots. Plots for a particular job (or project) are configured in the job configuration screen, where each field has additional help information. Each plot can have one or more lines (called data series). After each build completes the plots’ data series latest values are pulled from the CSV file generated by phploc._
  ```
  stage('Plot Code Coverage Report') {
    steps {
      plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Lines of Code (LOC),Comment Lines of Code (CLOC),Non-Comment Lines of Code (NCLOC),Logical Lines of Code (LLOC)                          ', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'A - Lines of code', yaxis: 'Lines of Code'
      plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Directories,Files,Namespaces', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'B - Structures Containers', yaxis: 'Count'
      plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Average Class Length (LLOC),Average Method Length (LLOC),Average Function Length (LLOC)', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'C - Average Length', yaxis: 'Average Lines of Code'
      plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Cyclomatic Complexity / Lines of Code,Cyclomatic Complexity / Number of Methods ', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'D - Relative Cyclomatic Complexity', yaxis: 'Cyclomatic Complexity by Structure'      
      plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Classes,Abstract Classes,Concrete Classes', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'E - Types of Classes', yaxis: 'Count'
      plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Methods,Non-Static Methods,Static Methods,Public Methods,Non-Public Methods', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'F - Types of Methods', yaxis: 'Count'
      plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Constants,Global Constants,Class Constants', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'G - Types of Constants', yaxis: 'Count'
      plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Test Classes,Test Methods', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'I - Testing', yaxis: 'Count'
      plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Logical Lines of Code (LLOC),Classes Length (LLOC),Functions Length (LLOC),LLOC outside functions or classes ', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'AB - Code Structure by Logical Lines of Code', yaxis: 'Logical Lines of Code'
      plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Functions,Named Functions,Anonymous Functions', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'H - Types of Functions', yaxis: 'Count'
      plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Interfaces,Traits,Classes,Methods,Functions,Constants', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'BB - Structure Objects', yaxis: 'Count'
    }
  }
  ```
- Install `phploc` on Jenkins server to successfully run the pipeline.
  ```
  sudo dnf --enablerepo=remi install php-phpunit-phploc -y
  wget -O phpunit https://phar.phpunit.de/phpunit-7.phar
  chmod +x phpunit
  sudo yum install php-xdebug
  ```
- Return to php-todo pipeline project on Jenkins Dashboard. Click on Scan Repository to sync and build. If successful, you should have `Plot` menu within project. Click to view plotted code analysis report.
- Add stage/step for bundling artifact for sending to artifactory. Ensure `zip` is installed on Jenkins server `sudo yum install zip -y`
  ```
  stage('Package Artifact') {
    steps {
      sh 'zip -qr php-todo.zip ${WORKSPACE}/*'
    }
  }
  ```
- Add stage/step for publishing artifact to artifactory.
  ```
  stage('Publish to Artifactory') {
    steps {
      script {
        def server = Artifactory.server 'artifactory-server' 
        def uploadSpec = """{
          "files": [
            {
              "pattern": "php-todo.zip",
              "target": "prjfourteen/php-todo",
              "props": "type=zip;status=ready"
            }
          ]
        }"""
        server.upload spec: uploadSpec
      }
    }
  }
  ```

## Deploy Todo App to Dev Environment
- Provision a server for `php-todo` app to be deployed. Ensure port 80 is open in Firewall
- Add server to `dev` inventory hosts in `ansible-config-mgt` project
- Add deployment assignment inside `ansible-config-mgt/static-assignments/deployment.yml` to 
- - Ensure Apache Webserver (`httpd`) is installed
- - Ensure `httpd` service is started and enabled
- - Ensure `php` is installed
- - Ensure php-fpm is started and enabled
- - Download the artifact from artifactory. Username and Password token must be provided. Password token can be generated in repository on artifactory dashboard.
- - Unzip the artifact
- - Copy the artifact to `/var/wwww/html/`
- - Remove default server welcome page
- - Restart `httpd` service
- Import assignment into the entry playbook `site.yml`
- In `php-todo` project add stage/step for deploying to server. This stage will trigger the `ansible-config-mgt` job in Jenkins.
  ```
  stage('Package Artifact') {
    steps {
      build job: 'ansible-project/main', parameters: [[$class: 'StringParameterValue', name: 'env', value: 'dev.yml']], propagate: false, wait: true
    }
  }
  ```
- Commit both projects to remote repositories.
- Return to Jenkins Dashboard. In `php-todo` project, scan repository and build job.


## Install & Configure SonarQube
- Provision a server for SonarQube. Should be a minimum of `4GB Memory` and `2 vCPUs`. Use the Ubuntu OS.
- Install SonarQube by using an Ansible Role. Use the following steps and configuration code to setup the role:
- - Create a role directory structure and name it `sonarqube` manually or use the `ansible-galaxy init sonarqube` command to create the role directory structure automatically. Alternatively, find a pre-configured role on Ansible Galaxy for installing SonarQube. Here I will use the role provided in this repo `https://github.com/Livingstone95/ansible-config/tree/main/roles/sonarqube`
- - Add sonarqube assignment inside `ansible-config-mgt/static-assignments/sonarqube.yml` to use the `sonarqube` role:
    ```yml
    ---
    - hosts: sonarqube
      become: true
      roles:
        - sonarqube
    ```
- - Import assignment into the entry playbook `site.yml`
- Commit code to remote repository.
- Return to Jenkins Dashboard. In `ansible-config-mgt` -> `main`, build with parameters and enter the `ci` environment.
- With SonarQube successfully installed, visit `<sonarqube-public-ip-address>:9000/sonar`
- Login to sonarqube dashboard with `admin` as both username and password.
- In Jenkins Dashboard install `SonarQube Scanner` plugin.
- Navigate to `Manage Jenkins` -> `Configure System`. Scroll to SonarQube Servers section.
- Add SonarQube server. Add a name. Enter URL to SonarQube installation `http://<sonarqube-public-ip-address>:9000/sonar` and save.
- Return to SonarQube Dashboard. Go to `Administration` -> `Configuration` -> `Webhooks`
- Click `Create` to create a new webhook. Enter a name. Enter `http://<jenkins-public-ip-address>/sonarqube-webhook/` in URL field. Click save.
- Return to Jenkins Dashboard to configure SonarQube Scanner Tool in `Manage Jenkins` -> `Configure Global Tool`.
- Scroll to SonarQube Scanner section. Click `Add SonarQube scanner`. Enter a name. Tick `Install automatically`
- Back in `php-todo` Jenkinsfile, add stage/step to add a SonarQube Scanner and Quality Gate to pipeline. Add this stage just before the `Package Artifact` stage. This way, the test can be done before the code is packaged and deployed.
  ```
  stage('SonarQube Quality Gate') {
    environment {
      scannerHome = tool 'SonarQubeScanner'
    }
    steps {
      withSonarQubeEnv('sonarqube') {
        sh "${scannerHome}/bin/sonar-scanner"
      }
    }
  }
  ```
- Commit to remote repository.
- Return to Jenkins Dashboard. In `php-todo` project, scan repository and build job.
- NOTE: This will fail!
- By running the build, Jenkins will install the scanner tool on the Linux server. You will need to go into the tools directory on the Jenkins server to configure the properties file in which SonarQube will require to function during pipeline execution.
  ```
  cd /var/lib/jenkins/tools/hudson.plugins.sonar.SonarRunnerInstallation/SonarQubeScanner/conf/
  ```
- Open the `sonar-scanner.properties` file.
  ```
  sudo vi sonar-scanner.properties
  ```
- Add the following configuration:
  ```
  sonar.host.url=http://<SonarQube-Server-IP-address>:9000/sonar/
  sonar.projectKey=php-todo
  #----- Default source code encoding
  sonar.sourceEncoding=UTF-8
  sonar.php.exclusions=**/vendor/**
  sonar.php.coverage.reportPaths=build/logs/clover.xml
  sonar.php.tests.reportPath=build/logs/junit.xml
  ```
- Return to Jenkins job and restart the build.


---


## PS _from Darey_

The recently implemented quality gate is ineffective. Why? Because upon visiting the SonarQube UI, you will discover that we have just pushed low-quality code to the development environment.

To begin, navigate to the SonarQube interface and access the "php-todo" project.

Upon inspection, you will notice the presence of bugs and a lack of code coverage, indicated by a 0.0% coverage rate. Code coverage represents the percentage of unit tests developed by programmers to assess the functionality of code functions and objects.

Clicking on the "php-todo" project for further analysis will reveal an accumulation of technical debt, code smells, and security issues within the codebase, totaling approximately six hours of effort.

While such issues may be acceptable in the development environment, as developers continually refine their code towards perfection, as a DevOps engineer responsible for the pipeline, it is crucial to ensure that the quality gate step triggers a pipeline failure when quality conditions are not met.

Conditional deployment to higher environments:

In the real world, developers typically work on feature branches within repositories such as GitHub or GitLab. Various branches are employed to control software releases effectively. Some common branch types include:

- Develop
- Master or Main
- Feature/*
- Release/*
- Hotfix/*
- And others

There exists a broad discussion around release strategies and git branching strategies, with GitFlow being a prominent approach. I encourage you to explore GitFlow further, as it may serve as a valuable topic for interview discussions.

Assuming a basic GitFlow implementation, only the "develop" branch is allowed to deploy code to the integration environment, such as the "sit" environment.

Let's update our Jenkinsfile to implement this:

Firstly, we'll include a "When" condition to trigger the Quality Gate only when the running branch matches "develop," "hotfix," "release," "main," or "master" using regular expressions:
```
when { branch pattern: "^develop*|^hotfix*|^release*|^main*", comparator: "REGEXP" }
```
Next, we add a timeout step that waits for SonarQube to complete its analysis and ensures the pipeline only succeeds if the code quality meets the required standards:
```
timeout(time: 1, unit: 'MINUTES') {
    waitForQualityGate abortPipeline: true
}
```
The complete stage will now appear as follows:
```
stage('SonarQube Quality Gate') {
    when { branch pattern: "^develop*|^hotfix*|^release*|^main*", comparator: "REGEXP" }
    environment {
        scannerHome = tool 'SonarQubeScanner'
    }
    steps {
        withSonarQubeEnv('sonarqube') {
            sh "${scannerHome}/bin/sonar-scanner -Dproject.settings=sonar-project.properties"
        }
        timeout(time: 1, unit: 'MINUTES') {
            waitForQualityGate abortPipeline: true
        }
    }
}
```
To test this, create different branches and push them to GitHub. You will observe that only branches other than "develop," "hotfix," "release," "main," or "master" will be able to deploy the code.


Note that with the current state of the code, it cannot be deployed to integration environments due to its substandard quality. In the real world, DevOps engineers would send this code back to the developers for further improvements based on the SonarQube quality report. Once the code quality meets the required standards, the pipeline will pass, enabling the code to proceed to higher environments.

---

# Extras
## Add new server as Jenkins slave
- Spinup new server
- Install Java
  ```
  sudo yum install java-11-openjdk-devel -y
  ```
- Open .bash_profile
  ```
  vi .bash_profile
  ```
- Insert Java Path
  ```
  export JAVA_HOME=$(dirname $(dirname $(readlink $(readlink $(which javac))))) 
  export PATH=$PATH:$JAVA_HOME/bin 
  export CLASSPATH=.:$JAVA_HOME/jre/lib:$JAVA_HOME/lib:$JAVA_HOME/lib/tools.jar
  ```
- Return to Jenkins Dashboard. Goto `Manage jenkins` -> `Manage Nodes` -> `New Node`
- Enter name of node. Select `Permanent Agent`. 
- Enter path to a directory on remote (slave) server such as user home directory `/home/user/`


## Configure GitHub Webhook with Jenkins
- Go to Github project for `php-todo` -> `Settings` -> `Webhook`
- In `Payload URL` enter `http://<jenkins-public-ip>:8080/github-webhook/`
- In `Content type` select `application/json`
- Click `Add Webhook`