#### Overview

> Let’s continue our demonstrate within the integrated case

- Pipeline was running that can create a feature branch on gitlab from our previous branch `v6.1-integratedCase-PUSH` 
- Now we can go ahead to our development process for fix bug or make new function to our application. Next let’s merge this feature into the `release branch`

#### Create pipeline
duplicate to the `new pipeline`

original pipeline: `demo-java-service_PUSH`
new pipeline: `demo-java-service_UAT`

- tick: This project is parameterized
    - Add `Choice Parameter`
        - Name: `srcUrl`
        - Choices: `http://10.4.7.101:8088/demo/demo-maven-service.git`
    - Add `String Parameter`
        - Name: `branchName`
        - Choices: `main`
    - Add `Choice Parameter`
        - Name: `buildType`
        - Choices: `mvn`
    - Add `String Parameter`
        - Name: `buildShell`
        - Default Value: `clean package -DskipTests`
- Remove Token option
- Script Path: `java.jenkinsfile`

The above `java.jenkinsfile` is in [this](https://github.com/galaxy-devops/jenkinsfiles) project

#### Docker image repository
We use aliyun remote image repository for quick building and uploading

##### Create namespace
Just create a specified namespace for our demonstrate `devopstest` in the image repository

##### Create credential
Need to create credential of login image repository on jenkins for subsequence pipeline. 

#### Operation on Jira
- Open and edit an existed issue
- Specify a release  in the `Fix Version/s`
- It will trigger pipeline running for building while doing press `done` button. A new release branch will be created on gitlab

 <img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v6/assignFixVersion-DEV10.png" style="zoom:67%;" />

- It also will create a MR from issue branch. Let’s initiate press `Merge` button manually.

 <img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v6/mergeBranch.png" style="zoom:67%;" />

#### Running pipeline
Let’s go ahead after pressing button. Just only `release branch` left here, so, we then start to build pipeline on Jenkins. That will achieve these target:

- Input the branch `RELEASE-1.6` then press `Build` button

- Pipeline script will generate docker image then deploy to the `UAT` environment. The pod are going to run the new image within `demo-uat` namespace

 <img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v6/jenkinsBuidReleaseBranch.png" style="zoom:67%;" />
