pipeline {
    agent any

    environment {
        IMAGE_NAME = 'deekshithsn/devops-training'
    }

    stages {
        stage('Get Git Commit as Docker Tag') {
            steps {
                script {
                    def tag = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    env.DOCKER_TAG = tag
                    currentBuild.displayName = "Final_Demo #${currentBuild.number} - ${tag}"
                }
            }
        }

        stage('Quality Gate Static Check') {
            agent {
                docker {
                    image 'maven'
                    args '-v $HOME/.m2:/root/.m2'
                }
            }
            steps {
                script {
                    withSonarQubeEnv('sonarserver') {
                        sh "mvn sonar:sonar"
                    }

                    timeout(time: 1, unit: 'HOURS') {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            error "Pipeline aborted due to quality gate failure: ${qg.status}"
                        }
                    }

                    sh "mvn clean install"
                }
            }
        }

        stage('Build Docker Image & Push') {
            steps {
                script {
                    sh 'cp -r ../devops-training@2/target .'
                    sh "docker build . -t ${IMAGE_NAME}:${DOCKER_TAG}"

                    withCredentials([string(credentialsId: 'docker_password', variable: 'DOCKER_PASSWORD')]) {
                        sh "docker login -u deekshithsn -p ${DOCKER_PASSWORD}"
                        sh "docker push ${IMAGE_NAME}:${DOCKER_TAG}"
                    }
                }
            }
        }

        stage('Run Ansible Deployment') {
            steps {
                script {
                    sh """
                        sed -i 's/docker_tag/${DOCKER_TAG}/g' deployment.yaml
                    """
                    ansiblePlaybook(
                        become: true,
                        installation: 'ansible',
                        inventory: 'hosts',
                        playbook: 'ansible.yaml'
                    )
                }
            }
        }
    }
}
