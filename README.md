### sonarQube

> In this branch, will demonstrate that pipeline invoke sonarQube to scan code via api

#### Quality profile
click `Quality Profiles` in the tab then click `Create`
- Name：`demo`
- Language：`java`
- Parent：`None`

 <img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v2/qualityProfiles-demo.png" style="zoom:75%;" />

let’s activate the rule in the above profile. Now, please click `Activate more` button

 <img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v2/qualityProfiles-rule.png" style="zoom:75%;" />

we can specify any rule which we want to apply in the window. Click `Activate` button to put rule into our profile.

 <img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v2/qualityProfiles-activeMore.png" style="zoom:75%;" />

Next step, we also need to configure a `severity level` to our profile. Choose a `level` then press `Activate` 

 <img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v2/qualityProfiles-severity.png" style="zoom:75%;" />

At last, let’s apply that profile to our app project. 

 <img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v2/qualityProfiles-set.png" style="zoom:75%;" />

#### Quality gate
create a quality gate and specify a metric value. Click `create` button in the tab of sonarqube.

- name: `demo`
- click `Add Condition`
    - tick：`On Overall Code`
    - Quality Gate fails when，input：`Code Smells`
    - Value：`1`

 <img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v2/qualityGates-demo.png" style="zoom:50%;" />

> The above value is just only for test.

Then, let’s associate this quality gate to our app project

 <img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v2/qualityGates-attach.png" style="zoom:50%;" />

As above the brief introduction that we can achieved the target of scanning our code by manipulation sonarQube manual. Next, let’s catch that purpose by code.

#### Jenkinslib
> prerequisite plugin httpRequest

go to  [jenkinslib](https://github.com/galaxy-devops/jenkinslib) repository . Open and edit this file: `sonarapi.groovy` to add these functions as below

##### Define HTTP request

```groovy
package org.devops

//encapulate HTTP request for sonarQube
def HttpReq(reqType,reqUrl,reqBody){
    /**  
     * @param reqType     request type, ex: http
     * @param reqUrl      API adress where to request
     * @param reqBody     request body
     */

    def sonarServer = "http://sonarqube:9000/api"

    result = httpRequest authentication: 'sonar-admin-user',
                httpMode: "${reqType}",
                contentType: "APPLICATION_JSON",
                consoleLogResponseBody: true,
                ignoreSslErrors: true,
                requestBody: "${reqBody}",
                url: "${sonarServer}/${reqUrl}",
                quiet: true
    return result
}
```

> Note:
>
> - `http://sonarqube:9000/api` , I installed sonarQube on my k8s platform, so this URL  is service name of k8s.
> - `sonar-admin-user`, this is credential id within jenkins, it shows that we need to set it before this function

##### Search project

```groovy
// search the project on the sonarQube
def SearchProject(projectName){

    /**
     * @param projectName  the project name
     */

    apiUrl = "projects/search?projects=${projectName}"
    response = HttpReq("GET",apiUrl,'')
    response = readJSON text: """${response.content}"""
    result = response["paging"]["total"]

    if (result.toString() == "0"){
        return "false"
    } else {
        return "true"
    }
}
```

##### Create project

```groovy
// create the project on sonarQube
def CreateProject(projectName){

    /**
     * @param projectName the project name which will be created
     */

    apiUrl = "projects/create?name=${projectName}&project=${projectName}"
    response = HttpReq("POST",apiUrl,'')
}
```

##### Query quality profile

```groovy
// get the quality profile
def GetQualityProfiles(qpname){

    /**
     * @param qpname the quality profile name
     */

    apiUrl = "qualityprofiles/search?qualityProfile=${qpname}"
    response = HttpReq("GET",apiUrl,'')
    response = readJSON text: """${response.content}"""

    if (response["profiles"].isEmpty()){
        return "error"
    } else {
        return "true"
    }
}
```

##### Configure quality profile

```groovy
// configure QP
def ConfigQualityProfiles(projectName,lang,qpname){

    /**
      * @param projectName the project name
      * @param lang        code language
      * @param qpname      the quality profile name
     */
    vName = GetQualityProfiles(qpname)
    if (vName == "error") {
        println("there is error in quality profile or does not exist")
        return "error"
    } else {
        apiUrl = "qualityprofiles/add_project?language=${lang}&project=${projectName}&qualityProfile=${qpname}"
        response = HttpReq("POST",apiUrl,'')

        return "true"
    }
}
```

##### Query quality gate

```groovy
// get QG name
def GetQualityGateName(gateName){

    /**
     * @param gateName the quality gate name
     */

    apiUrl = "qualitygates/show?name=${gateName}"
    response = HttpReq("GET",apiUrl,'')
    response = readJSON text: """${response.content}"""

    if (response["name"]) {
        return "true"
    } else {
        println("${gateName} not existed")
        return "error"
    }
}
```

##### Configure quality gate

```groovy
// configure QG
def ConfigQualityGates(projectName,gateName){

     /**
      * @param projectName the project name
      * @param gateName    the quality gate name
      */

    vName = GetQualityGateName(gateName)
    if (vName == "error") {
        println("there is error in quality gate or does not exist")
        return "error"
    } else {
        apiUrl = "qualitygates/select?gateName=${gateName}&projectKey=${projectName}"
        
        response = HttpReq("POST",apiUrl,'')
        return "true"
    }
}
```

##### Query status

```groovy
// get the status from sonarQube QG
def GetProjectStatus(projectName){

    /**
     * @param projectName the project name
     */

    // define API endpoint
    apiUrl = "project_branches/list?project=${projectName}"
    response = HttpReq("GET",apiUrl,'')
    response = readJSON text: """${response.content}"""
    result = response["branches"][0]["status"]["qualityGateStatus"]
    return result
}
```

##### Scan code

go to  [jenkinslib](https://github.com/galaxy-devops/jenkinslib) repository . The file `sonarqube.groovy`

```groovy
package org.devops

def SonarScan(sonarServer,projectName,projectDesc,projectPath){
    
    /**
     * @param sonarServer defined environment variable from Jenkins
     * @param projectName project name
     * @param projectDesc project description
     * @param projectPath project path where the endpoint of you project
     */

    // define server list in case test or prod situation
    def servers = ["test":"sonarqube-test","prod":"sonarqube-prod"]
    
    withSonarQubeEnv("${servers[sonarServer]}"){
        def scannerHome = "/usr/local/buildtools/sonar-scanner"
        def sonarDate = sh returnStdout: true, script: 'date +%Y%m%d%H%M%S'
        sonarDate = sonarDate - "\n"
        sh """
            ${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=${projectName} \
            -Dsonar.projectName=${projectName} \
            -Dsonar.projectVersion=${sonarDate} \
            -Dsonar.ws.timeout=30 \
            -Dsonar.projectDescription="${projectDesc}" \
            -Dsonar.sources=${projectPath} \
            -Dsonar.sourceEncoding=UTF-8 \
            -Dsonar.java.binaries=target/classes \
            -Dsonar.java.test.binaries=target/test-classes \
            -Dsonar.java.surefire.report=target/surefire-reports -X
        """
    }
}
```
Note:
We need to do the prerequisite for above block code

    - Prepare the token of sonarQube then add it to the Jenkins credential
    - Configure the sonarQube installations from Jenkins. I've already configured `sonarqube-test` setting

 <img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v2/sonarQube-servers.png" style="zoom:50%;" />


#### Jenkinsfile

let’s add `QA` stage in `ci.jenkinsfile` of [jenkinsfile](https://github.com/galaxy-devops/jenkinsfiles)

```groovy
stage("QA"){
    steps{
        script{
            tools.PrintMes("Searching project on sonarQube","green")
            result = sonarapi.SearchProject("${JOB_NAME}")

            if (result == "false") {
                println("${JOB_NAME} --- project does not exist, will create it --- ${JOB_NAME}")
                sonarapi.CreateProject("${JOB_NAME}")
            } else {
                println("${JOB_NAME} --- project exist")
            }
            tools.PrintMes("Associate QP for ${JOB_NAME} ","green")
            // our pipeline name is demo-maven-service_UAT
            // qpName = "demo"
            qpName = "${JOB_NAME}".split("-")[0]
            sonarapi.ConfigQualityProfiles("${JOB_NAME}","java",qpName)
            if (result == "error"){
                error "Got problem to quality profile"
            }                    
            tools.PrintMes("Associate QG for ${JOB_NAME}","green")
            result = sonarapi.ConfigQualityGates("${JOB_NAME}",qpName)
            if (result == "error"){
                error "Got problem to quality gate"
            }
            tools.PrintMes("Start to evaluate code with sonarQube ","green")
            sonar.SonarScan("test","${JOB_NAME}","${JOB_NAME}","src")
            tools.PrintMes("Get the result","green")
            result = sonarapi.GetProjectStatus("${JOB_NAME}")
            if (result.toString().contains("ERROR")){
                error "code have something wrong within quality gate, please check it"
            }else{
                println(result)
            }
        }
    }
}
```