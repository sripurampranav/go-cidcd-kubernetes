pipeline {
    agent any
    environment {
        PATH = "/usr/local/go/bin:$PATH"
        DOCKER_IMAGE_NAME = "chrisgreene/go-cicd-kubernetes"
    }
    stages {
        stage('Build') {
            steps {
                echo 'Running build automation'
                sh 'go test -v'
            }
        }
        stage('Build Docker Image') {
            when { 
                branch 'master'
            }
            steps {
                script {
                    app = docker.build(DOCKER_IMAGE_NAME)
                    app.withRun("-d -p 8181:8181") { c ->
                        sh 'curl localhost:8181'
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
                    docker.withRegistry('https://registry.hub.docker.com', 'docker_hub') {
                        app.push("${env.BUILD_NUMBER}")
                        app.push("latest")
                    }
                }
            }
        }
        stage('DeployToPKS') {
            when {
                branch 'master'
            }
            steps {
                //input 'Deploy to PKS?'
                milestone(1)
                withCredentials([usernamePassword(credentialsId: 'pks_client', usernameVariable: 'USERNAME', passwordVariable: 'USERPASS')]) {
                    script {
                        sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$pks_client \"kubectl delete all -l app=gocicd\""
                    }
                }
                kubernetesDeploy(
                  kubeconfigId: 'kubeconfig',
                  configs: 'kubernetes.yaml',
                  enableConfigSubstitution: true
                ) 
            }
        }
        stage('Get Service IP') {
            when {
                branch 'master'
            }
            steps {
                //milestone(1)
                retry(10) {
                    withCredentials([usernamePassword(credentialsId: 'pks_client', usernameVariable: 'USERNAME', passwordVariable: 'USERPASS')]) {
                        script {
                            //sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$pks_client \"kubectl get all -l app=gocicd -o wide\""
                            def ip = sh (script: "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$pks_client \"kubectl get svc gocicd --output=jsonpath={'.status.loadBalancer.ingress[].ip'}\"", returnStdout: true)
                            sh (sleep 5)
                            echo "IP is ${ip}"
                            echo "URL is http://${ip}:8181"
                            try {
                             //echo sh(script: "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$kube-master-ip ls -la", returnStdout: true)
                             //sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$kube-master-ip \"/usr/bin/touch /tmp/jenkins\""
                            } catch (err) {
                             echo: 'caught error: $err'
                            }
                        }
                    }
                }
            }
        }
    }
}
