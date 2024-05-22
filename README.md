#### Overview
We’ve already deployed `release` revision on UAT  environment in previous branch. Next, let’s go ahead to upgrade it to the `STAG` environment which is a promotion task

#### Initiate STAG
> Note:
>
> We need assure that the application is running meanwhile doing rely on these commands to quickly deploy the  application if it doesn’t exist before the subsequence process.

```bash
$ kubectl get deployment -n demo-uat -o yaml > test.yaml

$ sed -i 's/demo-uat/demo-stag/g' test.yaml

$ kubectl create -f test.yaml

$ kubectl create clusterrolebinding \
demoapp-stag-cluster-role-binding-admin \
--clusterrole=admin \
--serviceaccount=demo-stag:demoapp
```

#### Pipeline for promotion
> Promotion from UAT to STAG

Create a pipeline project, name：`demo-java-service_UPDATE`

- tick: `This project is parameterized`
    - Add `choice parameter`
        - Name: `updateType`
        - Choices: `UAT-to-STAG` and `STAG-to-PROD`
        - Description: `promotion type`
    - Add `string parameter`
        - Name: `releaseVersion`
        - Default Value: `null`
        - Description: `Specify revision`
- Pipeline:
    - Definition: `Pipeline script from SCM`
    - SCM: `git`
    - Repository URL: `http://10.4.7.101:8088/devops/jenkinsfiles.git`
    - Script Path: `promotion.jenkinsfile`

#### Running pipeline
Input the value in updateType：`UAT-to-STAG`
Input the value in releaseVersion：`1.6`

 <img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v6/pipelineBuilding.png" style="zoom:75%;" />

> Note:
>
> Let’s take `UAT-to-STAG` as example in our this branch

#### Validation

Go to the `demo-stag` directory in the gitlab, navigate file `1.6-stag.yaml` to check the `image` section after running pipeline

It has been turned into like this:

`registry.cn-beijing.aliyuncs.com/devopstest/demo-java-service:RELEASE-1.6`

 <img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v6/verify.png" style="zoom:80%;" />

