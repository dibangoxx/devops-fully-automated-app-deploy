def COLOR_MAP = [
    'SUCCESS': 'good',
    'FAILURE': 'danger',
]

pipeline {
    agent any

    environment {
        WORKSPACE = pwd()
        MAVEN_HOME = tool 'localMaven'
        JDK_HOME = tool 'localJdk'
        SONARQUBE_HOME = tool 'SonarQubeScanner'
    }

    stages {
        stage('Git checkout') {
            steps {
                echo 'Cloning the application code...'
                git branch: 'main', url: 'https://github.com/dibangoxx/devops-fully-automated-app-deploy.git'
            }
        }

        stage('Build') {
            steps {
                echo 'Building the application...'
                sh "echo 'MAVEN_HOME=${MAVEN_HOME}' >> ${WORKSPACE}/.env"
                sh "echo 'JDK_HOME=${JDK_HOME}' >> ${WORKSPACE}/.env"
                sh "${MAVEN_HOME}/bin/mvn -U clean package"
            }

            post {
                success {
                    echo 'Archiving artifacts...'
                    archiveArtifacts artifacts: '**/*.war', followSymlinks: false
                }
            }
        }

        stage('Unit Test') {
            steps {
                echo 'Running unit tests...'
                sh "${MAVEN_HOME}/bin/mvn test"
            }
        }

        stage('Integration Test') {
            steps {
                echo 'Running integration tests...'
                sh "${MAVEN_HOME}/bin/mvn verify -DskipUnitTests"
            }
        }

        stage('Checkstyle Code Analysis') {
            steps {
                echo 'Running Checkstyle analysis...'
                sh "${MAVEN_HOME}/bin/mvn checkstyle:checkstyle"
            }
            post {
                success {
                    echo 'Generated Checkstyle Analysis Result'
                }
            }
        }

        stage('SonarQube scanning') {
            steps {
                echo 'Running SonarQube analysis...'
                withSonarQubeEnv('SonarQube') {
                    withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
                        sh """
                            ${SONARQUBE_HOME}/bin/sonar-scanner \
                            -Dsonar.projectKey=maven \
                            -Dsonar.host.url=http://172.31.30.174:9000 \
                            -Dsonar.login=$SONAR_TOKEN
                        """
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                echo 'Waiting for Quality Gate...'
                waitForQualityGate abortPipeline: true
            }
        }

        stage('Upload artifact to Nexus') {
            steps {
                echo 'Uploading artifact to Nexus...'
                withCredentials([usernamePassword(credentialsId: 'nexus-credentials', passwordVariable: 'PASSWORD', usernameVariable: 'USER_NAME')]) {
                    sh """
                        cp ${WORKSPACE}/nexus-setup/settings.xml ${MAVEN_HOME}/conf/settings.xml
                        ${MAVEN_HOME}/bin/mvn clean deploy -DskipTests
                    """
                }
            }
        }

        stage('Deploy to Environment') {
            matrix {
                axes {
                    axis {
                        name 'ENVIRONMENT'
                        values 'dev', 'stage', 'prod'
                    }
                }
                stages {
                    stage('Deploy') {
                        steps {
                            echo "Deploying to ${ENVIRONMENT} environment..."
                            withCredentials([usernamePassword(credentialsId: 'ansible-deploy-server-credentials', passwordVariable: 'PASSWORD', usernameVariable: 'USER_NAME')]) {
                                sh """
                                    ansible-playbook -i ${WORKSPACE}/ansible-setup/aws_ec2.yaml ${WORKSPACE}/deploy.yaml \
                                    --extra-vars "ansible_user=$USER_NAME ansible_password=$PASSWORD hosts=tag_Environment_${ENVIRONMENT} workspace_path=$WORKSPACE"
                                """
                            }
                        }
                    }
                }
            }
        }

        stage('Approval') {
            steps {
                input('Do you want to proceed?')
            }
        }
    }

    post {
        always {
            echo 'Sending notification to Slack...'
            slackSend channel: '#team-devops', color: COLOR_MAP[currentBuild.currentResult], message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}"
        }
    }
}
