#### Overview
> this branch's target is to show the share library then utilize it in the fundamental pipeline.


#### Share library
Navigate to the `Manage Jenkins`/`System Configuration`/`System` in the dashboard of jenkins. Click `Add` button in the `Global Pipeline Libraries`
- Name: `jenkinslib`
- Default version：`master`
- Retrieval method：`Modern SCM`
- Source Code Management：`Git`
- Project Repository：`http://10.4.7.101:8088/devops/jenkinslib.git`
- Credentials：`gitlab-admin-user`
- Library Path (optional)：`./`

 <img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v1/jenkinslib.png" style="zoom:50%;" />

> 1. `10.4.7.101:8088/devops/jenkinslib.git`  is our internal repository on gitlab
> 2. Prepare it before this setting
> 3. the Credentials option is not the mandatory requirement
> 4. The above `jenkinslib` is also in [github](https://github.com/galaxy-devops/jenkinslib) 

#### Apply share library
>  prerequisite plugin in jenkins: AnsiColor

Add the file `tools.groovy` in the [jenkins shared library](https://github.com/galaxy-devops/jenkinslib)

```groovy
package org.devops

//print content
def PrintMes(value,color){
    colors = ['red': "\033[40;31m >>>>>>>>>>${value}<<<<<<<<<< \033[0m",
              "blue": "\033[47;34m ${value} \033[0m",
              "green": "\033[40;32m >>>>>>>>>>${value} <<<<<<<<<< \033[0m" ]
    ansiColor('xterm') {
        println(colors[color])
    }
}
```

Then, create a pipeline project in Jenkins. Paste the below content in the `pipeline section`

```groovy
#!groovy
@Library('jenkinslib') _
def tools = new org.devops.tools()
pipeline {
    agent any

    stages {
        stage("GetCode"){
            steps{          
                timeout(time: 5, unit: 'MINUTES'){
                    script{                           
                        println('get the code')
                        tools.PrintMes("Retrieve the code", 'red')
                    }
                }
            }
        }
        stage("Build"){
            steps{
                timeout(time: 20, unit: 'MINUTES'){
                    script{
                        println('packet')
                        tools.PrintMes("packet the code", 'green')
                    }
                }
            }
        }
        stage("CodeScan"){
            steps{
                timeout(time: 30, unit: 'MINUTES'){
                    script{
                        println("scan code")
                        tools.PrintMes("this is my ansiColor test", 'blue')
                    	}
                	}
            	}
        	}
    	}
    }
```
Then, we could start to build this pipeline to test our `tools` 

#### Definition maven

##### Setting
Navigate to the `Manage Jenkins`/`System Configuration`/`Tools` in the dashboard of jenkins. Click `Add` button in the `Maven installations`

- Name：`M2`
- Don't tick：`Install automatically`
- MAVEN_HOME：`/usr/local/buildtools/apache-maven`

 <img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v1/maven.png" style="zoom:80%;" />

##### Pipeline

> Let's test maven setting what if working properly in the pipeline

Create pipeline
- name：`demo-mavenci-service`
- Definition：`Pipeline Script from SCM`
    - SCM：git
    - Repository URL：`http://10.4.7.101:8088/devops/jenkinsfiles.git`
    - Credentials：`gitlab-admin-user`
    - Branch Specifier：`*/master`
    - Script Path：`ci.jenkinsfile`

> 1. `10.4.7.101:8088/devops/jenkinsfiles.git`  is our internal repository on gitlab
> 2. Prepare it before this pipeline
> 3. Credentials option is not the mandatory requirement
> 4. The above `ci.jenkinsfile` is also in [github](https://github.com/galaxy-devops/jenkinsfiles) 

##### Test
make a file: `ci.jenkinsfile`  in the repository.

```groovy
#!groovy
pipeline{
    agent any
    stages{
        stage("build"){
            steps{
                script{
                    mvnHome = tool "M2"
                    sh "${mvnHome}/bin/mvn -v"
                }
            }
        }
    }
}
```

start to build the above pipeline

#### Integration maven
##### Create file

File: `build.groovy` in the [jenkinslib](https://github.com/galaxy-devops/jenkinslib) 

```groovy
package org.devops

//building
def Build(buildType,buildShell){

    /**
     * @param   buildType       the value from the choice parameter in jenkins
     * @param   buildShell      the command from the jenkins
     */

    def buildTools=["mvn":"M2","ant":"ANT","gradle":"GRADLE","npm":"NPM"]

    println("The type of building is ${buildType}")

    buildHome = tool buildTools[buildType]

    if ("${buildType}" == "npm"){
        sh """
            export NODE_HOME=${buildHome}
            export PATH=\$NODE_HOME/bin:\$PATH
            ${buildHome}/bin/${buildType} ${buildShell}
           """
    }else{
        sh "${buildHome}/bin/${buildType} ${buildShell}"
    }
}
```

Edit the file: `ci.jenkinsfile`  in the [jenkinsfiles](https://github.com/galaxy-devops/jenkinsfiles)

```groovy
#!groovy

@Library('jenkinslib') _

def build = new org.devops.build()

String buildType = "${env.buildType}"
String buildShell = "${env.buildShell}"

pipeline{
    agent any
    stages{
        stage('build'){
            steps{
                script{
                    build.Build(buildType,buildShell)
                }
            }
        }
    }
}
```

##### Create pipeline
Create a pipeline in `jenkins` then set the option as below:

- tick: `This project is parameterized`
  - click `Add Parameter`
    - Add `Choice parameter`
      - Name：`buildType`
      - Choices：
        - mvn
        - ant
        - gradle
        - npm
      - Description：`build type`
  
     <img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v1/integratedMavenTool-1.png" style="zoom:50%;" />

  - click `Add Parameter`
    - Add `Choice Parameter`
      - Name：`buildShell`
      - Choices：
        - -v
        - clean
        - clean package
        - clean install
        - clean test
      - Description：`building command`

     <img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v1/integratedMavenTool-2.png" style="zoom:50%;" />

Then click `Build` button to start building

 <img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v1/integratedMavenTool-3.png" style="zoom:50%;" />

#### Provision project
> project URL: https://github.com/galaxy-devops/demo-maven-service

##### App
- Create `group` and name it `demo` 
- Create a repository `demo-maven-service`  under the group `demo`
- upload code

##### On jenkins

Create a pipeline item, name: `demo-maven-service_PUSH`
tick `This project is parameterized`
- click `Add parameter`
  - Add：`choice parameter`
    - Name：`srcUrl`
    - Choices：`http://10.4.7.101:8088/demo/demo-maven-service.git`

     <img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v1/prepareProject-srcUrl.png" style="zoom:50%;" />

- click `Add parameter`
  - Add：`choice parameter`
    - Name：`branchName`
    - Choices：`master`

     <img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v1/prepareProject-branchName.png" style="zoom:50%;" />

- click `Add parameter`
  - Add：`choice parameter`
    - Name：`buildType`
    - Choices：`mvn`

     <img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v1/prepareProject-buildType.png" style="zoom:50%;" />

- click `Add parameter`
  - Add：`String parameter`
    - Name：`buildShell`
    - Default Value：` clean package -DskipTests`

     <img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v1/prepareProject-buildShell.png" style="zoom:50%;" />

in `Pipeline section`
- Definition：`Pipeline script SCM`
  - SCM：`Git`
  - Repository URL：`http://10.4.7.101:8088/devops/jenkinsfiles.git`
  - Branch Specifier：`*/master`
  - Script Path：`ci.jenkinsfile`

##### On gitlab
go to`jenkinsfiles` repository

edit file：`ci.jenkinsfile`

>  Add the environment variable

- String buildType = "${env.buildType}"
- String buildShell = "${env.buildShell}"
- String srcUrl = "${env.srcUrl}"
- String branchName = "${env.branchName}"

Create a stage on the top

```groovy
#!groovy

@Library('jenkinslib') _

def build = new org.devops.build()
def tools = new org.devops.tools()

String buildType = "${env.buildType}"
String buildShell = "${env.buildShell}"
String srcUrl = "${env.srcUrl}"
String branchName = "${env.branchName}"

// pipeline
pipeline{
    agent any
    stages{
        stage('CheckOut'){
            steps{
                script{
                    tools.PrintMes("Retrieve the code","green")
                    checkout([$class: 'GitSCM', branches: [[name: "${branchName}"]], extensions: [], userRemoteConfigs: [[url: "${srcUrl}"]]])
                }
            }
        }

        stage('Build'){
            steps{
                script{
                    tools.PrintMes("Start to building","green")
                    build.Build(buildType,buildShell)
                }
            }
        }
    }
}
```

##### Building

>  start to build pipeline

 <img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v1/prepareProject-startBuilding.png" style="zoom:50%;" />

#### Supplement

##### Configure webhook

> prerequisite plugin: Generic Webhook Trigger

open our above pipeline item：`demo-maven-service_PUSH`

tick：`Generic Webhook Trigger`

- Request parameter
  
- Name of request parameter：`runOpts`
  
- Token：`demo-maven-service_PUSH`

    <img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v1/webhook.png" style="zoom:50%;" />

>  go to repository`demo-maven-service` in gitlab

go to the `webhooks` in the `settings`

- URL：`http://10.4.7.102:8080/generic-webhook-trigger/invoke?token=demo-maven-service_PUSH&runOpts=GitlabPush`
- Secret token: `null`
- tick：`Push events`
- keep default value to the rest items

 <img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v1/webhook-2.png" style="zoom:50%;" />

##### Branch matching

open the pipeline item `demo-maven-service_PUSH` in the jenkins

click `Add` button for variable in `Post content parameters`
- Name of variable：`branch`
- Expression：`$.ref`
- tick：`JSONPath`

 <img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v1/branchMatch-1.png" style="zoom:75%;" />

edit the file: `ci.jenkinsfile` then add the lines 

```groovy
if ("${runOpts}" == "GitlabPush"){
                        branchName = branch - "refs/heads/"
                    }
```

```groovy
stages{
        stage('CheckOut'){
            steps{
                script{
                    tools.PrintMes("Retrieve the code","green")
                    if ("${runOpts}" == "GitlabPush"){
                        branchName = branch - "refs/heads/"
                    }
                    tools.PrintMes("The branch is ${branchName}","green") 
                    checkout([$class: 'GitSCM', branches: [[name: "${branchName}"]], 
                    extensions: [], 
                    userRemoteConfigs: [[url: "${srcUrl}"]]])
                }
            }
        }
```

##### Attach description information

edit file：`ci.jenkinsfile`

```groovy
stages{
        stage('CheckOut'){
            steps{
                script{
                    tools.PrintMes("Retrieve the code","green")
                    if ("${runOpts}" == "GitlabPush"){
                        branchName = branch - "refs/heads/"
                        currentBuild.description = "Trigger by ${userName} and branch is ${branchName}"
                    }
                    tools.PrintMes("The branch is ${branchName}","green") 
                    checkout scmGit(branches: [[name: "${branchName}"]], extensions: [], userRemoteConfigs: [[credentialsId: 'gitlab-credentials', url: "${srcUrl}"]])
                }
            }
        }
```

##### Attach post stage

edit file：`ci.jenkinsfile`

```groovy
    post {
        always{
            script{
                println("THIS IS always in post")
            }
        }

        success{
            script{
                println("success")
            }
        }

        failure{
            script{
                println("failure")
            }
        }

        aborted{
            script{
                println("aborted")
            }
        }
    }
```

This is the finally whole content of file `ci.jenkinsfile`
```groovy
#!groovy

@Library('jenkinslib') _

def build = new org.devops.build()
def tools = new org.devops.tools()

//define this variable to avoid exception whenever manually build
def runOpts

String buildType = "${env.buildType}"
String buildShell = "${env.buildShell}"
String srcUrl="${env.srcUrl}"
String branchName="${env.branchName}"

//if true then indicate that from webhook
//else to show that via manipulation by human being
if ("${runOpts}" == "GitlabPush"){
    branchName = branch - "refs/heads/"
    currentBuild.description = "Trigger by ${userName} ${branchName}"
} else {
    currentBuild.description = "Manually trigger building on ${branchName}"
}

// pipeline
pipeline{
    agent any
    stages{
        stage('CheckOut'){
            steps{
                script{
                    tools.PrintMes("Retrieve the code","green")
                    if ("${runOpts}" == "GitlabPush"){
                        branchName = branch - "refs/heads/"
                    }
                    tools.PrintMes("The branch is ${branchName}","green")               
                    checkout([$class: 'GitSCM', branches: [[name: "${branchName}"]], extensions: [], userRemoteConfigs: [[url: "${srcUrl}"]]])
                }
            }
        }

        stage('Build'){
            steps{
                script{
                    tools.PrintMes("Start to building","green")
                    build.Build(buildType,buildShell)
                }
            }
        }
    }
    post {
        always{
            script{
                println("THIS IS always in post")
            }
        }
        success{
            script{
                println("success")
            }
        }
        failure{
            script{
                println("failure")
            }
        }
        aborted{
            script{
                println("aborted")
            }
        }
    }
}
```