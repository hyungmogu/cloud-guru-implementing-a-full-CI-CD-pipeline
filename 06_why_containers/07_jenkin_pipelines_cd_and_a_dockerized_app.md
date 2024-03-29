# Jenkin Pipelines CD and a Dockerized App

## Imagining Dockerized Deployment Process

1. Jenkin produces a Docker image with the new code
2. Jenkin pushes the new image to docker hub
3. Jenkin pulls the new image on the production server
4. Jenkin stops the container running the old code
5. Jenkin runs docker run on the server to run the new image

<img src="https://user-images.githubusercontent.com/6856382/225777966-eb086d48-8b13-408f-83bb-a7137f9eda00.png" />


## Requirements for Jenkin - Docker CI/CD

### 1) Storing Production Server IP using Environmental Variable

- Jenkins allows the storage of environmental valriables
- Jenkins environmental variables are storked in `Manage Jenkins` > `Configure System` > `Global Properties`

<img src="https://user-images.githubusercontent.com/6856382/225778461-da5b4c2b-f4ab-44a3-9d8b-ea9444c897a4.png"/>

<img src="https://user-images.githubusercontent.com/6856382/225778500-667e1f23-107c-449f-9af5-f79e18ff280a.png"/>

### 2) Adding Credentials

- At least 3 credentials are required:
    1. webserver_login
        - this is the key used to login to each server using ssh
    2. docker_hub_login
        - this is the key required to login to docker hub
    3. github_key
        - this is the key required to pull, push, watch repository on github

<img src="https://user-images.githubusercontent.com/6856382/225778987-cc5885b8-7c84-4fec-865f-6192bda22954.png"/>

### Docker plugin

### 3) Build Pipeline
- The source code of the repository is found [here](https://github.com/linuxacademy/cicd-pipeline-train-schedule-dockerdeploy)
- The following build pipeline is used to deploy app to server using docker

```
pipeline {
    agent any
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
                    app = docker.build("willbla/train-schedule")
                    app.inside {
                        sh 'echo $(curl localhost:8080)'
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
        stage('DeployToProduction') {
            when {
                branch 'master'
            }
            steps {
                input 'Deploy to Production?'
                milestone(1)
                withCredentials([usernamePassword(credentialsId: 'webserver_login', usernameVariable: 'USERNAME', passwordVariable: 'USERPASS')]) {
                    script {
                        sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$prod_ip \"docker pull willbla/train-schedule:${env.BUILD_NUMBER}\""
                        try {
                            sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$prod_ip \"docker stop train-schedule\""
                            sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$prod_ip \"docker rm train-schedule\""
                        } catch (err) {
                            echo: 'caught error: $err'
                        }
                        sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$prod_ip \"docker run --restart always --name train-schedule -p 8080:8080 -d willbla/train-schedule:${env.BUILD_NUMBER}\""
                    }
                }
            }
        }
    }
}
```


## Note

- `sshpass` is required in both jenkins server and the production server for jenkin to use use password to login to production server