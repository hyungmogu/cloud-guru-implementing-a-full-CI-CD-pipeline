# Fully Automated Deployment

- `Fully Automated Deployment` needs confidence in 
    1. Automation
    2. Testing
    3. Monitoring

- `Fully Automated Deployment` is not desirable in all cases, but in some cases, it can be great

## Instructions - Setting up Fully Automated Deployment

1. On Jenkins web browser page, click `Manage Jenkins` > `Configure System`

2. Scroll down to `Global Properties` and select `Environmental Variables`

<img src="https://user-images.githubusercontent.com/6856382/226923545-c5ab6fbc-0445-44a7-88a3-bcb8df7a820d.png">

3. Add environment variable `KUBE_MASTER` and it's ip address

<img src="https://user-images.githubusercontent.com/6856382/227123499-a2ae64fc-54ea-435a-adad-dcab39104844.png">

4. Scroll down to `Github` section and turn on Github Webhook trigger
5. Select `Github Server` after clicking `Add Github Server` dropdown

<img src="https://user-images.githubusercontent.com/6856382/227125381-96cf1e87-21d1-4011-997c-32d56a22b5e9.png">

6. Add following information to the field:
    - `Name`: Github
    - `API URL`: http://api.github.com
    - `Manage hooks`: selected

6. Add another github credential as done previously
- But this time, set it under the kind `secret text`

<img src="https://user-images.githubusercontent.com/6856382/227126548-4804089c-1678-4545-91cb-1ebfc10116a3.png">
<img src="https://user-images.githubusercontent.com/6856382/227126926-545fda1b-74c2-4c6e-adcf-62efa62a3712.png">

8. Click `save`

7. Click `Manage Jenkins` > `Manage Plugin` and Download `httpRquest` plugin
- This allows to make http requests inside pipeline
- This will be used for smoke tests

<img src="https://user-images.githubusercontent.com/6856382/227395672-8800b9fc-1566-4717-bf7d-7c0670d6770a.png">

8. In `Jenkinsfile`, make sure to replace `willbla` with your personal docker hub repository ID
- `milestone(1)` forces all prior builds to go through in order before proceeding. 
- `post build actions` allow steps to be invoked regardless of status of Jenkin pipeline.
- `CANARY_REPLICAS = 0` deletes deployment from Kubernetes cluster


**Jenkinsfile**
```
pipeline {
    agent any
    environment {
        //be sure to replace "willbla" with your own Docker Hub username
        DOCKER_IMAGE_NAME = "willbla/train-schedule"
        CANARY_REPLICAS = 0
    }
    stages {
        stage('Build') {
            steps {
                echo 'Running build automation'
                sh './gradlew build --no-daemon'
                archiveArtifacts artifacts: 'dist/trainSchedule.zip'
            }
        }
        stage('Build Docker Image') {
            when {
                branch 'master'
            }
            steps {
                script {
                    app = docker.build(DOCKER_IMAGE_NAME)
                    app.inside {
                        sh 'echo Hello, World!'
                    }
                }
            }
        }
        stage('Push Docker Image') {
            when {
                branch 'master'
            }
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker_hub_login') {
                        app.push("${env.BUILD_NUMBER}")
                        app.push("latest")
                    }
                }
            }
        }
        stage('CanaryDeploy') {
            when {
                branch 'master'
            }
            environment { 
                CANARY_REPLICAS = 1
            }
            steps {
                kubernetesDeploy(
                    kubeconfigId: 'kubeconfig',
                    configs: 'train-schedule-kube-canary.yml',
                    enableConfigSubstitution: true
                )
            }
        }
        stage('SmokeTest') { // <- THIS GUY HERE
            when {
                branch 'master'
            }
            steps {
                script {
                    sleep (time: 5)
                    def response = httpRequest (
                        url: "http://$KUBE_MASTER_IP:8081/",
                        timeout: 30
                    )
                    if (response.status != 200) {
                        error("Smoke test against canary deployment failed.")
                    }
                }
            }
        }
        stage('DeployToProduction') {
            when {
                branch 'master'
            }
            steps {
                milestone(1)
                kubernetesDeploy(
                    kubeconfigId: 'kubeconfig',
                    configs: 'train-schedule-kube.yml',
                    enableConfigSubstitution: true
                )
            }
        }
    }
    post {
        cleanup {
            kubernetesDeploy (
                kubeconfigId: 'kubeconfig',
                configs: 'train-schedule-kube-canary.yml',
                enableConfigSubstitution: true
            )
        }
    }
}
```

9. On Jenkins browser home page, create `Multibranch Pipeline` with name `train-schedule`
- Set `credential` to `github_key` created in earlier steps
- Set `Owner` to `your github id`
- Set `Repository` to `cicd-pipeline-train-schedule-autodeploy`

<img src="https://user-images.githubusercontent.com/6856382/227399657-ef89c3fa-ec0b-44f0-bb0e-a9598893642b.png">

10. Select `Okay` and run

11. Validate the setup of Kubernetes deployment via running the following line of command

**Kubernetes Cluster**
```
kubectl get pods -w
```

