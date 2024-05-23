#### Overview
Our new release version have already been deployed to the `STAG` for now. Then next, we just repeat our early steps to deploy it to the `PROD` which is `STAG-to-PROD` just like in early branch. I skip that process in this branch and our target as so far:

- Describe the publication for  release revision to master branch
- Summary the whole steps

#### Publish revision

- Go to `dashboard` in the `Project/DEVOPS` on Jira
- Go to the menu of `Releases` 
- Click `Release` action of current revision `1.6` 
   <img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v6.5/v6.5-publishRelease-1.png" style="zoom:85%;" />
  - Ignore and proceed with release: keep it
  - Release date: 
    - Click `select a date` then `today`
- Click `Release`

> Note:
>
> - This publication operation will trigger pipeline to create MR. Merge `release branch` into `master branch` 

#### Summary steps

##### Step I - Create release

We create release while getting request of bug fix or development to new function on Jira.  For example, we input `1.7` in `Version name` of `Release` menu on Jira. 

- It will trigger `jira-devops-service` pipeline.
- Pipeline script will retrieve `deployment` resource object in the `demo-uat` namespace of kubernetes.
- It spawn a `YAML` file(1.7.yaml) from `demoapp-uat-deploy` deployment.
- Then, upload the file to gitlab, repository name: `demo-info-service`
- File path: `demo-info-service/demo-uat/1.7.yaml`

The branch `v5-yaml-edition-management` describe these above steps. We use this snippet in our `jira.jenkinsfile`

```groovy
stage("CreateVersionFile"){
    when {
        environment name: 'eventType', value: 'jira:version_created'
    }
    steps{
        script{
            println("Get the file from Deployment")
            response = k8s.GetDeployment("demo-uat","demoapp-uat-deploy")
            response = response.content

            println("Upload the file")
            base64Content = response.bytes.encodeBase64().toString()
            gitlab.CreateRepoFile(6,"demo-uat%2f${versionName}.yaml",base64Content)                    
        }
    }
}
```

##### Step II - Create issue

Create an issue on Jira dashboard. We specify these parameter

- Project: keep it
- Issue Type:  `Task`
- Summary: `DEV-11` 
- Component/s: `demo-java-service`
- Fix Version/s: `Don't assign any release`

Then it trigger pipeline to create a branch `DEV-11` on gitlab. The branch `v6.1-integratedCase-PUSH` describe above steps. We use this snippet in our `jira.jenkinsfile`

```groovy
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
```

##### Step III - Development

Next, proceed to develop to bug fix or new function on the new branch `DEV-11` then commit the effort. It trigger pipeline to build code/scan/upload artifact. Pipeline use `gitlab.jenkinsfile`

##### Step IV - Association

Let’s associate to the release after previous step. Open the issue on Jira then specify the above release(1.7)

- It will trigger pipeline to create a MR(`DEV-11` to `RELEASE-1.7`)
- Manually merge it on gitlab

<img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v6.5/v6.5-step4-merge.png" style="zoom:80%;" />

The branch `v6.2-integratedCase-UAT` describe above steps. We use this snippet in our `jira.jenkinsfile`

```groovy
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
```

##### Step V - Deployment

The next, let’s deploy branch `RELEASE-1.7` to `UAT`  within pipeline. Input `RELEASE-1.7` in the input field of pipeline

<img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v6.5/v6.5-step5-UAT.png" style="zoom:80%;" />

The pipeline will make the effort:

- Update image. It will update image in `demo-info-service/demo-uat/1.7.yaml`
- New deployment in `demo-uat` namespace on kubernetes

The branch `v6.2-integratedCase-UAT` describe these above steps. Pipeline use `java.jenkinsfile`

##### Step VI - Promotion

We then next to upgrade our app in `UAT`. Proceed to build pipeline

- choose `UAT-to-STAG`
- releaseVersion: `1.7`

<img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v6.5/v6.5-step6-UPGRADE.png" style="zoom:80%;" />



It create file on gitlab. New file `demo-info-service/demo-stag/1.7-stag.yaml`

The branch `v6.3-integratedCase-UPDATE` describe these above steps. Pipeline use `promotion.jenkinsfile`   

##### Step VII - Stag

Deployment `release revision` to `STAG`. Let’s go ahead  our pipeline to deploy this new `release version` to `STAG` namespace.

- stackName: `STAG`
- releaseVersion: `1.7`



<img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v6.5/v6.5-step7-deploy.png" style="zoom:80%;" />



It update image of deployment on kubernetes

```bash
 $ kubectl describe po demoapp-stag-deploy-588b74878c-ndxnf -n demo-stag | grep Image:
    Image:      registry.cn-beijing.aliyuncs.com/devopstest/demo-java-service:RELEASE-1.7
```

The branch `v6.4-integratedCase-DEPLOY` describe the detail in above steps. Pipeline use `deploy.jenkinsfile`

It’s just like above two `Step VI` and `Step VII` that we could repeat them for `PROD` in pipeline

- choose `STAG-to-PROD`
- releaseVersion: `1.7`

then deploy it to `PROD`

- stackName: `PROD`
- releaseVersion: `1.7`

I will skip these detail in here.

##### Step VIII - publish

The last step, let’s publish our release version if app running stable for a while. Click the  `Release` action of `1.7` item in the `Release` menu on the Jira. For example, like this as below

<img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v6.5/v6.5-publishRelease-1.png" style="zoom:80%;" />

That operation will trigger pipeline to create MR which  `RELEASE-1.7` request merge into `master` branch. `RELEASE-1.7` branch would be purged after doing merge is accomplished.

Pipeline use `stage("Publish release version")` in file `jira.jenkinsfile`

