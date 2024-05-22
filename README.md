#### Overview
The app’s image was upgraded after building pipeline in previous branch `v6.3-integratedCase-UPGRADE` . So we’re going to next step to `deploy` it on the `STAG` environment in this branch.

#### Create pipeline
Create the application deployment pipeline, name: `demo-java-service_DEPLOY`

- Tick `This project is parameterized`
    - Add `Choice Parameter`
        - Name: `stackName`
        - Choices: `one item in each line`
            - UAT
            - STAG
            - PROD
        - Description: where to deploy
    - Add `String Parameter`
        - Name: `releaseVersion`
        - Default Value: `none`
        - Description: input release version
- Pipeline
    - Definition: Pipeline script from SCM
    - SCM: git
    - Repository URL: `http://10.4.7.101:8088/devops/jenkinsfiles`
    - Branch Specifier: `master`
    - Script Path: `deploy.jenkinsfile`

#### Build pipeline
Initiate to build pipeline like it below

- stackName: we specify `STAG` which meaning the new version of image will deploy to stag
- releaseVersion: this is the `Fix/Version` value in Jira

 ![](https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v6/deploy.png)

#### Validation
Let’s check it on the kubernetes
```bash
$ kubectl get rs -n demo-stag
NAME                             DESIRED   CURRENT   READY   AGE
demoapp-stag-deploy-57444fd9cb   0         0         0       22d
demoapp-stag-deploy-5b567f8f46   1         1         1       30s
```
Here we can get a new `rs` object in kubernetes. Let’s check its `image`
```bash
$ kubectl describe po/demoapp-stag-deploy-5b567f8f46-6t87b -n demo-stag | grep Image:
    Image:         registry.cn-beijing.aliyuncs.com/devopstest/demo-java-service:RELEASE-1.6
```
