### Overview
This branch’s target is to achieve the managed file which is resource object of kubernetes on the gitlab in case that we also can fulfil the version control to file due to replace/remove image of pod

#### Create repository
Create repo on the gitlab
- name: `demo-info-service`

Create respectively directory for each environment

 <img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v5/revisionRepository.png" style="zoom:75%;" />

#### Update Jira webhook 
Navigate to `webhook`  and `update option` which we’ve already chosen them before on jira. Tick the `created` in the Version

 <img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v5/webhook-1.png" style="zoom:75%;" />

#### Create resource object

> Note:
>
>  We need to prepare the resource object on kubernetes in advance. We execute these steps if it doesn’t exist.

##### Create namespace
Running command `kubectl` to create respective namespace

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: demo-uat
---
apiVersion: v1
kind: Namespace
metadata:
  name: demo-stag
---
apiVersion: v1
kind: Namespace
metadata:
  name: demo-prod
```

```bash
$ kubectl create -f ns.yaml
```

##### Create deployment
> We take nginx as example in this branch

###### UAT

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  labels:
    k8s-app: demoapp-uat
  name: demoapp-uat-deploy
  namespace: demo-uat
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: demoapp-uat
  template:
    metadata:
      labels:
        k8s-app: demoapp-uat
      namespace: demo-uat
      name: demoapp-uat
    spec:
      containers:
        - name: demoapp-uat
          image: nginx
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8081
              name: web
              protocol: TCP
      serviceAccountName: demoapp-uat
---
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: demoapp-uat
  name: demoapp-uat-svc
  namespace: demo-uat
spec:
  type: NodePort
  ports:
    - name: web
      port: 8081
      targetPort: 8081
      nodePort: 30991
  selector:
    k8s-app: demoapp-uat
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: demoapp-uat
  name: demoapp-uat
  namespace: demo-uat
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
 name: demoapp-uat
 namespace: demo-uat
rules:
 - apiGroups: [""]
   resources: ["pods","configmaps","namespaces"]
   verbs: ["create","delete","get","list","patch","update","watch"]
 - apiGroups: [""]
   resources: ["pods/exec"]
   verbs: ["create","delete","get","list","patch","update","watch"]
 - apiGroups: [""]
   resources: ["pods/log"]
   verbs: ["get","list","watch"]
 - apiGroups: [""]
   resources: ["secrets"]
   verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
 name: demoapp-uat-role-binding
 namespace: demo-uat
roleRef:
 apiGroup: rbac.authorization.k8s.io
 kind: Role
 name: demoapp-uat
subjects:
 - kind: ServiceAccount
   name: demoapp-uat
   namespace: demo-uat
```

```bash
$ kubectl create -f demoapp-uat.yaml
$ kubectl get deploy -n demo-uat
$ kubectl get po -n demo-uat
$ kubectl get service -n demo-uat
```

###### STAG
```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  labels:
    k8s-app: demoapp-stag
  name: demoapp-stag-deploy
  namespace: demo-stag
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: demoapp-stag
  template:
    metadata:
      labels:
        k8s-app: demoapp-stag
      namespace: demo-stag
      name: demoapp-stag
    spec:
      containers:
        - name: demoapp-stag
          image: nginx
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8081
              name: web
              protocol: TCP
      serviceAccountName: demoapp-stag
---
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: demoapp-stag
  name: demoapp-stag-svc
  namespace: demo-stag
spec:
  type: NodePort
  ports:
    - name: web
      port: 8081
      targetPort: 8081
      nodePort: 30992
  selector:
    k8s-app: demoapp-stag
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: demoapp-stag
  name: demoapp-stag
  namespace: demo-stag
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
 name: demoapp-stag
 namespace: demo-stag
rules:
 - apiGroups: [""]
   resources: ["pods","configmaps","namespaces"]
   verbs: ["create","delete","get","list","patch","update","watch"]
 - apiGroups: [""]
   resources: ["pods/exec"]
   verbs: ["create","delete","get","list","patch","update","watch"]
 - apiGroups: [""]
   resources: ["pods/log"]
   verbs: ["get","list","watch"]
 - apiGroups: [""]
   resources: ["secrets"]
   verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
 name: demoapp-stag-role-binding
 namespace: demo-stag
roleRef:
 apiGroup: rbac.authorization.k8s.io
 kind: Role
 name: demoapp-stag
subjects:
 - kind: ServiceAccount
   name: demoapp-stag
   namespace: demo-stag
```

```bash
$ kubectl create -f demoapp-stag.yaml
$ kubectl get po -n demo-stag
$ kubectl get svc -n demo-stag
```

###### PROD
```yaml
---
kind: Deployment
apiVersion: apps/v1
metadata:
  labels:
    k8s-app: demoapp-prod
  name: demoapp-prod-deploy
  namespace: demo-prod
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: demoapp-prod
  template:
    metadata:
      labels:
        k8s-app: demoapp-prod
      namespace: demo-prod
      name: demoapp-prod
    spec:
      containers:
        - name: demoapp-prod
          image: nginx
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8081
              name: web
              protocol: TCP
      serviceAccountName: demoapp-prod
---
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: demoapp-prod
  name: demoapp-prod-svc
  namespace: demo-prod
spec:
  type: NodePort
  ports:
    - name: web
      port: 8081
      targetPort: 8081
      nodePort: 30993
  selector:
    k8s-app: demoapp-prod
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: demoapp-prod
  name: demoapp-prod
  namespace: demo-prod
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
 name: demoapp-prod
 namespace: demo-prod
rules:
 - apiGroups: [""]
   resources: ["pods","configmaps","namespaces"]
   verbs: ["create","delete","get","list","patch","update","watch"]
 - apiGroups: [""]
   resources: ["pods/exec"]
   verbs: ["create","delete","get","list","patch","update","watch"]
 - apiGroups: [""]
   resources: ["pods/log"]
   verbs: ["get","list","watch"]
 - apiGroups: [""]
   resources: ["secrets"]
   verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
 name: demoapp-prod-role-binding
 namespace: demo-prod
roleRef:
 apiGroup: rbac.authorization.k8s.io
 kind: Role
 name: demoapp-prod
subjects:
 - kind: ServiceAccount
   name: demoapp-prod
   namespace: demo-prod
```

```bash
$ kubectl create -f demoapp-prod.yaml
$ kubectl get deploy -n demo-prod
$ kubectl get po -n demo-prod
$ kubectl get svc -n demo-prod
```

#### Create function
##### Retrieve token
```bash
$ kubectl describe secrets -n kube-system \
$(kubectl get secret -n kube-system | awk '/admin-user/{print $1}') | grep token: | awk '{print $2}'
```

> NOTE:
>
> we also need to add a credentials item in jenkins after this token

##### Get deployment 

> This section will dump current resource object and upload to gitlab while doing click `add` in the jira release

go to [jenkinslib](https://github.com/galaxy-devops/jenkinslib)

create the file: `kubernetes.groovy`

```groovy
package org.devops

//encapulse http request
def HttpReq(reqType,reqUrl,reqBody){
    
    /**
     * @param   reqType     the request type, such like http
     * @param   reqUrl      the url of server
     * @param   reqBody     the request body which to be sent
     */
       
    //k8s server
    def apiServer = "https://10.138.181.3:6443/apis/apps/v1"
    
    withCredentials([string(credentialsId: 'kubernetes-token', variable: 'kubernetestoken')]) {
        result = httpRequest customHeaders: [[maskValue: true, name: 'Authorization', value: "Bearer ${kubernetestoken}"],
                                            [maskValue: false, name: 'Content-Type', value: 'application/yaml'],
                                            [maskValue: false, name: 'Accept', value: 'application/yaml']],
                httpMode: "${reqType}",
                consoleLogResponseBody: true,
                ignoreSslErrors: true,
                requestBody: "${reqBody}",
                url: "${apiServer}/${reqUrl}",
                quiet: true
    }
    return result
}

// retrieve deployment resource object
def GetDeployment(nameSpace,deployName){
    
    /**
     * @param   nameSpace       the namespace in the k8s
     * @param   deployName      deployment object name
     */
    
    apiUrl = "namespaces/${nameSpace}/deployments/${deployName}"
    response = HttpReq('GET',apiUrl,'')
    return response
}
```
go to  [jenkinsfiles](https://github.com/galaxy-devops/jenkinsfiles)
open file:  `jira.jenkinsfile` then add more stage

```groovy
def k8s = new org.devops.kubernetes()
def gitlab = new org.devops.gitlab()

stage("CreateVersionFile"){
    when {
        environment name: 'eventType', value: 'jira:version_created'
    }
    steps{
        script{
            println("Get the file from Deployment")
            response = k8s.GetDeployment("demo-uat","demoapp-uat-deploy")
            response = response.content
        }
    }
}
```

In above block:

-  `jira:version_created` means the event type while doing create `release` on the Jira
- That will trigger the pipeline running
- Let’s take `demo-uat` namespace as example

##### upload file to gitlab

dump the resource object into the file then upload it to gitlab

go to [jenkinslib](https://github.com/galaxy-devops/jenkinslib)
edit file：`gitlab.groovy` then add more function

```groovy
// create file
def CreateRepoFile(projectId,filePath,fileContent){
    /**
     * @param   projectId       the project ID
     * @param   filePath        file path
     * @param   fileContent     file content which need to put on the gitlab
     */

    apiUrl = "projects/${projectId}/repository/files/${filePath}"
    reqBody = """{"branch": "master","encoding":"base64","content":"${fileContent}","commit_message":"create a new file"}"""
    response = HttpReq('POST',apiUrl,reqBody)
}
```
go to  [jenkinsfiles](https://github.com/galaxy-devops/jenkinsfiles)
add more in file：`jira.jenkinsfile`

```groovy
def k8s = new org.devops.kubernetes()
def gitlab = new org.devops.gitlab()

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

#### Merger 
> This section will trigger pipeline  while publish `release` on the Jira
>
> We also need prepare credential(jira-admin-user) in Jenkins

go to [jenkinslib](https://github.com/galaxy-devops/jenkinslib)

new file:  `jira.groovy` then add function

```groovy
package org.devops

// encapulate http request
def HttpReq(reqType,reqUrl,reqBody){
    
    /**
     * @param   reqType     the request type
     * @param   reqUrl      the request address
     * @param   reqBody     the request content
     */

    // jira server
    def apiServer = "http://10.138.181.3:18009/rest/api/2"
    result = httpRequest authentication: 'jira-admin-user',
            httpMode: "${reqType}",
            contentType: "APPLICATION_JSON",
            consoleLogResponseBody: true,
            ignoreSslErrors: true,
            requestBody: "${reqBody}",
            url: "${apiServer}/${reqUrl}",
            quiet: true
    
    return result

}

// send request to Jira
def RunJql(jqlContent){
    
    /**
     * @param   jqlContent      the command content
     */

    apiUrl = "search?jql=${jqlContent}"
    response = HttpReq("GET",apiUrl,'')
    return response
}
```
go to [jenkinsfiles](https://github.com/galaxy-devops/jenkinsfiles)

open file: `jira.jenkinsfile`

add new stage in case need publish `release` that will trigger pipeline.

```groovy
def jira = new org.devops.jira()

stage('Release Version'){
    when {
        environment name: 'eventType', value: 'jira:version_released'
    }

    steps{
        script{
            println("project%20%3D%20${projectKey}%20AND%20fixVersion%20%3D%20${versionName}%20AND%20issuetype%20%3D%20Task")
            response = jira.RunJql("project%20%3D%20${projectKey}%20AND%20fixVersion%20%3D%20${versionName}%20AND%20issuetype%20%3D%20Task")
            response = readJSON text: """${response.content}"""
            println(response)
        }
    }
}
```

>  Note:
>
> - The above println means: project = ${projectKey} AND fixVersion = ${versionName} AND issuetype = Task
> - We will add more content into this block in later branch

#### Create version on Jira

Let’s create a new version in Jira

- Go to `Projects/DEVOPS` then click `Releases` menu on the left of Jira
- Input a version, for example: `1.1.4`, then click `Add` button

<img src="https://gitee.com/galaxy-devops/image-pre-demo-spring-boot-service/raw/master/v5/v5-createRelease.png" style="zoom:75%;" />

