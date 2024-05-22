### Overview

Let’s demonstrate the usage of jira and the integration of it within pipeline in this branch

#### Working flow
1. Create `release` in jira
2. Create issue then associate with component
3. Jira will trigger pipeline process in case to create issue. Pipeline will create the branch  in gitlab
4. It will be triggered by association release during issue editing. It will create a release branch then generate a merge request

#### Integration to Jenkins
##### Configure webhook
- Navigate to the setting and click the `System`

 <img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v4/webhook-1.png" style="zoom:75%;" />

- Click `create` in the `Webhook`
    - Name：`jenkins-hook`
    - Status：`Enabled`
    - URL：`http://10.4.7.102:8080/generic-webhook-trigger/invoke?token=jira-devops-service&projectKey=${project.key}`
    - Events Issue related events
        - input this expression ：`project=DEVOPS`
        - tick theses ones in the issue：`created updated deleted` 
        - In the `Project related events`, tick： `released` in the `Version`
    - Click `create`
    
     <img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v4/webhook-2.png" style="zoom:75%;" />

##### Pipeline
Generate a pipeline project, name:`jira-devops-service`

- tick：`Generic Webhook Trigger`

- In `Post content parameters` , tick `Add`
    - Variable：`webHookData`
    - Expression：`$`
    - tick：`JSONPath`

- In `Request parameters` , tick `Add`
    - Request paramete：`projectKey`

- Token：`jira-devops-service`
- Pipeline：
    - Definition：`Pipeline script from SCM`
    - SCM：`Git`
    - Repository URL：`http://10.4.7.101:8929/devops/jenkinsfiles.git`
    - Branch Specifier: `*/master`
    - Script Path：`jira.jenkinsfile`

##### Jenkinsfile
In the [jenkinslib](https://github.com/galaxy-devops/jenkinslib) project
Edit file：`gitlab.groovy` and let’s add some function.

```groovy
package org.devops

// encapsulate HTTP request
def HttpReq(reqType,reqUrl,reqBody){
    
    /**
     * @param reqType       request type
     * @param reqUrl        API url which want to access
     * @param reqBody       request body
     */
    
    // gitlab server
    def gitServer = "http://10.4.7.101:8088/api/v4"
    withCredentials([string(credentialsId: 'gitlab-token', variable: 'gitlabToken')]) {
        result = httpRequest customHeaders: [[maskValue: true, name: 'PRIVATE-TOKEN', value: "${gitlabToken}"]],
                    httpMode: "${reqType}",
                    contentType: "APPLICATION_JSON",
                    consoleLogResponseBody: true,
                    ignoreSslErrors: true,
                    requestBody: "${reqBody}",
                    url: "${gitServer}/${reqUrl}",
                    quiet: true
    }
    return result
}

// retrieve ID
def GetProjectID(repoName='',projectName){

    /**
     * @param   repoName        the repository name
     * @param   projectName     the project name   
     */

    projectAPI = "projects?search=${projectName}"
    response = HttpReq('GET',projectAPI,'')
    def result = readJSON text: """${response.content}"""

    for (repo in result){
        if (repo['path'] == "${projectName}"){
            repoId = repo['id']
            println("Project ID：${repoId}")
            break
        }
    }
    return repoId
}


// create branch
def CreateBranch(projectId,refBranch,newBranch){

    /**
     * @param   projectId       the ID number of project
     * @param   refBranch       branch name which based on it
     * @param   newBranch       new branch name
     */

    try{
        branchApi = "projects/${projectId}/repository/branches?branch=${newBranch}&ref=${refBranch}"
        response = HttpReq("POST",branchApi,'').content
        branchInfo = readJSON text: """${response}"""
    } catch(e){
        println(e)
    }
}

// create branch request
def CreateMr(projectId,sourceBranch,targetBranch,title,assigneeUser=""){
    
    /**
     * @param       projectId       the id number of project which in gitlab
     * @param       sourceBranch    the source branch in MR
     * @param       targetBranch    the target branch which merge to
     * @param       title           the title info
     * @param       assigneeUser    the assignee name
     */
    
    try{
        def mrUrl = "projects/${projectId}/merge_requests"
        def reqBody = """{"source_branch":"${sourceBranch}","target_branch":"${targetBranch}","title":"${title}","assignee_id":"${assigneeUser}"}"""
        response = HttpReq("POST",mrUrl,reqBody).content
        return response
    } catch(e){
        println(e)
    }
}
```
> Note:  The plugin method in above block `withCredentials([string(credentialsId: 'gitlab-token', variable: 'gitlabToken')])`
>
> - It refer to the credential `gitlab-token` of Jenkins
> - We need to create that before this function

In the [`jenkinsfiles`](https://github.com/galaxy-devops/jenkinsfiles) project
edit file：`jira.jenkinsfile`

```groovy
#!groovy

@Library('jenkinslib') _

def gitlab = new org.devops.gitlab()

pipeline{
    agent any
    stages{
        stage("FilterData"){
            steps{
                script{
                    response = readJSON text: """${webHookData}"""
                    tools.PrintMes("Get the event type","green")
                    env.eventType = response["webhookEvent"]
                    println("Jira event type is：${eventType}")

                    switch(eventType) {
                        case "jira:issue_created":
                            env.issueName = response['issue']['key']
                            env.userName = response['user']['name']
                            env.moduleNames = response['issue']['fields']['components']
                            env.fixVersion = response['issue']['fields']['fixVersions']
                            currentBuild.description = "${eventType} - ${issueName} Trigger by ${userName} on Jira"
                            break
                        case "jira:issue_updated":
                            env.issueName = response['issue']['key']
                            env.userName = response['user']['name']
                            env.moduleNames = response['issue']['fields']['components']
                            env.fixVersion = response['issue']['fields']['fixVersions']
                            currentBuild.description = "${eventType} - ${issueName} Trigger by ${userName} on Jira"
                            break
                        case "jira:version_created":
                            env.versionName = response["version"]["name"]
                            currentBuild.description = "${eventType} - ${versionName} Trigger by Jira"
                            break
                        case "jira:version_released":
                            env.versionName = response["version"]["name"]
                            currentBuild.description = "${eventType} - ${versionName} Trigger by Jira"
                            break
                        default:
                            println(">>>>>>    unknown event    <<<<<<")
                            break
                    }
                }
            }
        }
        stage("Create issue branch or MR"){
            when {
                anyOf {
                    environment name: 'eventType', value: 'jira:issue_created'
                    environment name: 'eventType', value: 'jira:issue_updated'
                }
            }
            steps{
                script{
                    def projectIds = []
                    fixVersion = readJSON text: """${fixVersion}"""

                    def projects = readJSON text: """${moduleNames}"""
                    for (project in projects) {
                        projectName = project["name"]
                        currentBuild.description += "\n project: ${projectName}"
                        repoName = projectName.split("-")[0]
                        try {
                            projectId = gitlab.GetProjectID(repoName, projectName)
                            projectIds.add(projectId)
                        } catch(e){
                            println(e)
                        }
                    }
                    println("list the project ID：${projectIds}")

                    if (fixVersion.size() == 0){
                        for (id in projectIds) {
                            println("Create feature branch, project ID： ${id} issue： ${issueName}")
                            currentBuild.description += "\n New feature branch--> ${issueName}"
                            gitlab.CreateBranch(id,"main","${issueName}")
                        }
                    } else {
                        fixVersion = fixVersion[0]['name']
                        currentBuild.description += "\n Associate issue to release ,will generate MR \n ${issueName} --> RELEASE-${fixVersion}"

                        for (id in projectIds) {
                            println("Create branch RELEASE-${fixVersion}")
                            gitlab.CreateBranch(id,"main","RELEASE-${fixVersion}")
                            println("Generate MR ${issueName} ---> RELEASE-${fixVersion}")
                            gitlab.CreateMr(id,"${issueName}","RELEASE-${fixVersion}","${issueName}--->RELEASE-${fixVersion}")
                        }
                    }
                }
            }
        }
    }
}
```

>  stage `FilterData`

This stage route the traffic base on the event type from jira. It initialize respective variable in each different situation.



>  stage `Create issue branch or MR` 

This stage will handle what if need only to create issue/feature branch or create MR of issue branch to release branch. The `fixVersion` in this stage means `release` and it utilize the method `.size()` to reflect what if the issue associate to the release, so there are two option in here:

- if we specify a `fixVersion/s` in the issue window. That means the current issue branch will be merged into   the target `fixVersion` branch. It will create a  MR
- if we don’t specify a `fixVersion/s` in issue editing window. It will only just create a branch which name is identical with our issue.

