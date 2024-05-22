### Overview
Let’s demonstrate the usage of artifactory in this branch

#### Installation
1. Install JFrog Artifactory, it’s beyond the scope of this branch, Please refer to its [website](https://jfrog.com/help/r/jfrog-installation-setup-documentation/installing-artifactory) for detail
2. We also need to add credentials within Jenkins. It’s helpful in the subsequent part of this article.

#### Configure
>  prerequisite plugin: Artifactory

Navigate to `System`/`JFrog` of `Manage Jenkins`  then click `Add JFrog Platform Instance` button

 <img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v3/addJfrogPlatformInstance.png" style="zoom:75%;" />

- tick：`Use the Credentials Plugin`
- Instance ID：`artifactory`
- JFrog Platform URL：`http://10.138.181.3:8081`
- Default Deployer Credentials
    - Credentials: `artifactory-admin-user`
- click `Advanced Configuration` button
    - tick：`Bypass HTTP proxy`

- click `Test Connection` button to test connection

 <img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v3/jfrogSetting-1.png" style="zoom:75%;" />

will appear：`Found JFrog Artifactory 7.77.7 at http://10.138.181.3:8081/artifactory`

and ：`JFrog Distribution not found at http://10.138.181.3:8081/distribution`

 <img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v3/jfrog-connection.png" style="zoom:100%;" />

#### Naming conventions&Repository
##### Naming conventions
This is the hierarchical structure in artifact store

- Repository name
    - business/project name
        - application name
            - revision
                - artifact
##### Create repository
Let’s create the artifact repository base on the above rule in JFrog. For example like this as below

```text
repository name: demo-dev
project name: demo
application: demo-myapp-service
revision: 1.1-SNAPSHOT
artifact: demo-myapp-service-1.1-202001108.130358-1-4.jar
```

 ![](https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v3/namingConventions.png)

create these repository from JFrog：
- demo
- demo-dev
- demo-uat
- demo-stag
- demo-prod

 <img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v3/jfrog-localRepo.png" style="zoom:85%;" />

#### pipeline&jenkinsfile
> prerequisite plugin: rtUpload

##### jenkinsfile
create file: `artifactory.groovy`  in the [jenkinslib](https://github.com/galaxy-devops/jenkinslib) 

> NOTE:
>
> - this file’s target is to achieve building pipeline and upload artifact to the store
> - I just take maven as example to show the building and upload function

```groovy
package org.devops

// maven building
def MavenBuild(buildShell){

    /**
     * @param   buildShell  the build command from pipeline
     */

    def server = Artifactory.newServer url: "http://10.138.181.4:31084/artifactory"
    def rtMaven = Artifactory.newMavenBuild()
    def buildInfo = Artifactory.newBuildInfo()
    server.connection.timeout = 300
    server.credentialsId = 'artifactory-admin-user'

    rtMaven.tool = 'M2'

    String newBuildShell = "${buildShell}".toString()
    rtMaven.run pom: 'pom.xml', goals: newBuildShell, buildInfo: buildInfo
    server.publishBuildInfo buildInfo   
}

// upload artifact
def PushArtifact(){
    
    // get jar file
    def jarName = sh returnStdout: true, script: "cd target; ls *.jar"
    jarName = jarName - "\n"

    def pom = readMavenPom file: 'pom.xml'

    env.pomVersion = "${pom.version}"
    env.serviceName = "${JOB_NAME}".split("_")[0]
    env.buildTag = "${BUILD_ID}"

    // rename new jar file
    def newJarName = "${serviceName}-${pomVersion}-${buildTag}.jar"
    println("will rename jar file：${jarName} to the new one：${newJarName}")
    sh "mv target/${jarName} target/${newJarName}"

    // upload artifact
    env.businessName = "${env.JOB_NAME}".split("-")[0]
    env.repoName = "${businessName}-${JOB_NAME.split("_")[-1].toLowerCase()}"

    println("The artifact will be uploaded to ${repoName} ")
    env.uploadDir = "${repoName}/${businessName}/${serviceName}/${pomVersion}"

    println(">>>>>>>>>    start to upload    <<<<<<<<<")
    rtUpload (
        serverId: "artifactory",
        spec:
            """{
                "files": [
                    {
                        "pattern": "target/${newJarName}",
                        "target": "${uploadDir}/"
                    }
                ]
            }"""
    )
}

def main(buildType,buildShell){
    if (buildType == "mvn"){
        MavenBuild(buildShell)
    }
}
```
In above function

```
    String newBuildShell = "${buildShell}".toString()
    rtMaven.run pom: 'pom.xml', goals: newBuildShell, buildInfo: buildInfo
    server.publishBuildInfo buildInfo
```

it convert the type of argument into the string representing. Then rtMaven object’s `run` method read the `pom.xml` file from the project and transport relevant information to the variable `buildInfo` . At last the method `publishBuildInfo` would upload information from buildInfo to the remote artifact store.



In above function

```
    def jarName = sh returnStdout: true, script: "cd target; ls *.jar"
    jarName = jarName - "\n"

    def pom = readMavenPom file: 'pom.xml'

    env.pomVersion = "${pom.version}"
    env.serviceName = "${JOB_NAME}".split("_")[0]
    env.buildTag = "${BUILD_ID}"

    def newJarName = "${serviceName}-${pomVersion}-${buildTag}.jar"
    println("will rename jar file：${jarName} to the new one：${newJarName}")
    sh "mv target/${jarName} target/${newJarName}"
    
    // upload artifact
    env.businessName = "${env.JOB_NAME}".split("-")[0]
    env.repoName = "${businessName}-${JOB_NAME.split("_")[-1].toLowerCase()}"
    env.uploadDir = "${repoName}/${businessName}/${serviceName}/${pomVersion}"
```

this section will `rename` the primitive `tar` file to the new name which following our naming convention. Let me show an example from a building history of pipeline console output

- jarName: my-app-1.1-SNAPSHOT.jar
- JOB_NAME: demo-myapp2-service_DEV
- BUILD_ID: 10
- pomVersion: 1.1-SNAPSHOT
- serviceName: demo-myapp2-service
- buildTag: 10
- newJarName: demo-myapp2-service-1.1-SNAPSHOT-10.jar
- businessName: demo
- repoName: demo-dev
- uploadDir: demo-dev/demo/demo-myapp2-service/1.1-SNAPSHOT

##### pipeline
Create a pipeline project
pipeline name：`demo-maven-service_DEV`
tick：`This project is parameterized`

- add `Choice Parameter`
    - Name：`srcUrl`
    - Choices：`http://10.4.7.101:8088/demo/demo-maven-service.git`
- add `Choice Parameter`
    - Name：`branchName`
    - Choices：`main`
- add `Choice Parameter`
    - Name：`buildType`
    - Choices：`mvn`
- add `String Parameter`
    - Name：`buildShell`
    - Default Value：`clean package -DskipTests`
- Pipeline
    - Definition
      
        - choose：`Pipeline script from SCM`
        
            - SCM：`Git`
                - Repository URL：`http://10.4.7.101:8088/devops/jenkinsfiles.git`
                - Credentials: `specify a credential`
                - Branch Specifier：`*/master`
                - Script Path ：`ci.jenkinsfile`

   <img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v3/jfrog-building.png" style="zoom:30%;" />

The [jenkinsfile](https://github.com/galaxy-devops/jenkinsfiles) for pipeline

open and **<u>replace</u>**  early stage into this new one as below in file `ci.jenkinsfile` 

```groovy
stage('build'){
    steps{
        script{
            tools.PrintMes("Start to build pipeline ","green")
            artifactory.main(buildType,buildShell)

            tools.PrintMes("Upload artifact to store","green")
            artifactory.PushArtifact()
        }
    }
}
```