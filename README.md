#### Overview
Let’s demonstrate the whole steps within case in the series branches. In this branch, we will achieve these targets as below:

- Will create a demo application
- Create a pipeline project
- Configure a webhook to trigger pipeline to build application

#### Create the demo case
Let’s create the [case](https://github.com/galaxy-devops/demo-maven-service) on gitlab: `demo-java-service`

#### Create webhook
- Create the `webhook` in gitlab setting
- input: `http://10.4.7.102:8080/generic-webhook-trigger/invoke?token=demo-maven-service_PUSH&runOpts=GitlabPush`
- input `DEV*` in the `Push events`

#### Create pipeline project
Create the pipeline project, name: `demo-java-service_PUSH`
tick: This project is parameterized

- Add `Choice Parameter`
    - Name: srcUrl
    - Choices: `http://10.4.7.101:8088/pre-demo/demo-java-service.git`
- Add `Choice Parameter`
    - Name: branchName
    - Choices: main
- Add `Choice Parameter`
    - Name: buildType
    - Choices: `mvn`
- Add `String Parameter`
    - Name: buildShell
    - Default Value: `clean package -DskipTests`
- tick: Build Triggers
    - tick: `Generic Webhook Trigger`
    - Add `Post content parameters`
        - Name of variable: `branch`
        - Expression: `$.ref`
        - tick `JSONPath`
    - Add `Post content parameters`
        - Name of variable: `userName`
        - Expression: `$.user_username`
        - tick `JSONPath`
    - Add `Post content parameters`
        - Name of variable: `projectId`
        - Expression: `$.project.id`
        - tick `JSONPath`
    - Add `Post content parameters`
        - Name of variable: `before`
        - Expression: `$.before`
        - tick `JSONPath`
    - Add `Post content parameters`
        - Name of variable: `after`
        - Expression: `$.after`
        - tick `JSONPath`
    - Add `Post content parameters`
        - Name of variable: `object_kind`
        - Expression: `$.object_kind`
        - tick `JSONPath`
    - Request parameters
        - click `Add`
            - Name of request parameter: `runOpts`
            - Value filter: keep null
    - Token: `demo-java-service_PUSH`
    - keep default value in the rest part
    - Pipeline:
        - Definition: `Pipeline script from SCM`
        - SCM: `git`
        - Repository URL: `http://10.4.7.101:8088/devops/jenkinsfiles.git`
        - Credentials: choose existed one or leave it 
        - Branches to build：
            - Branch Specifier: `*/master`
        - Script Path输入：`gitlab.jenkinsfile`

#### Create craft store
Go to `Artifactory` and click `gear icon`(settings). Then create the repository which the type is `Local` in the `Repositoires` menu
- Repository Key: `demo-push`
- Environments: `DEV`

 <img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v6/artifactory-demoPUSH.png" style="zoom:75%;" />

#### Create feature branch
Go to project `DEVOPS`  in Jira then click `Issues menu` on left to create a issue.
- Project：`DEVOPS`
- Issue Type: `Task`
- Summary: `DEV-9`
- Reporter: `admin`
- Component/s：`demo-java-service`
- click `create` button

 <img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v6.1/v6.1-PUSH.png" style="zoom:75%;" />

> NOTE:
>
> - This operation will trigger the jira-devops-service pipeline and invocate the jira.jenkinsfile. Then it will create a feature branch which name `DEV-9` on the gitlab
>   - We input the specified branch name `DEV-9` in the input field which being followed the expression of webhook in gitlab
>   - We also must input the identical name in `Component/s` field which same with our gitlab repository
> - I didn’t specify the value in the `Fix Version/s` to associate current issue in above dialog box. So, it won’t create a MR to the release branch

#### On gitlab
##### Check new branch
Go to project  `demo-java-service` 

In the `Branches` menu, check what if the feature branch is existed. `DEV-9`

 <img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v6/branch-DEV9.png" style="zoom:75%;" />

##### On new feature branch
1. Checkout the branch `DEV-9`
2. Edit any  file in the branch then commit your change
3. It then will trigger webhook to invoke the pipeline project `demo-java-service_PUSH` running to build application/scan code/upload craft to store