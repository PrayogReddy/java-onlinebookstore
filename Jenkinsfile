pipeline {
    agent { label "${params.environment}" }
    parameters {
        choice choices: ['dev', 'qa', 'uat', 'prod'], description: 'Choose the env', name: 'environment'
        booleanParam defaultValue: true, description: 'Enable to run sonar scan', name: 'sonar'
        
    }
    
    environment {
        SONARQUBE_ENV = 'sonarscanner' 
        PROJECT_KEY = 'onlinebookstore'
        JFROG_URL = 'http://44.202.248.54:8081/artifactory/onlinebookstore/'
    }

    stages {
        stage('checkout') {
            steps {
                checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[credentialsId: '', url: 'https://github.com/PrayogReddy/java-onlinebookstore.git']])
            }
        }
        stage('sonarscan') {
            when {
                expression { params.sonar }
            }
            steps {
                withSonarQubeEnv("${SONARQUBE_ENV}") {
                    sh "mvn clean verify sonar:sonar -Dsonar.projectKey=${PROJECT_KEY}"
                    sh "sleep 10s"
                }
            }
        }
        stage('quality gate') {
            when {
                expression { params.sonar }
            }
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }
        stage('build') {
            steps {
                sh 'mvn clean package'
                archiveArtifacts artifacts: 'target/*.war', fingerprint: true
            }
            post {
                always {
                    jacoco execPattern: 'target/*.exec'
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }
        stage('push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'jfrog_creds', passwordVariable: 'password', usernameVariable: 'username')]) {
                    sh "curl -X PUT -u ${username}:${password} -T target/*.war '${JFROG_URL}/onlinebookstore-0.0.1-SNAPSHOT.war'"
                }
            }
        }
        stage('deploy') {
            steps {
                sh 'cp -r /home/ubuntu/jenkins/workspace/sonar_jfrog_pipeline/target/*.war /opt/tomcat/apache-tomcat-9.0.68/webapps/'
            }
        }
    }

    post {
    success {
        slackSend(
            channel: '#jenkins-notifications',
            color: 'good',
            message: """✅ *BUILD SUCCESSFUL*

            🚀 *Job:* ${env.JOB_NAME}
            🔢 *Build:* <${env.BUILD_URL}|#${env.BUILD_NUMBER}>
            🌍 *Environment:* ${params.environment}

            🎉 Pipeline completed successfully!
            📋 <${env.BUILD_URL}console|View Console Logs>""",
                        failOnError: false
                )
            }

    failure {
        slackSend(
            channel: '#jenkins-notifications',
            color: 'danger',
            message: """❌ *BUILD FAILED*

            🔧 *Job:* ${env.JOB_NAME}
            🔢 *Build:* <${env.BUILD_URL}|#${env.BUILD_NUMBER}>
            🌍 *Environment:* ${params.environment}

            ⚠️ Pipeline failed. Please check the logs.
            📋 <${env.BUILD_URL}console|Investigate Failure>""",
                        failOnError: false
                )
            }

    aborted {
        slackSend(
            channel: '#jenkins-notifications',
            color: 'warning',
            message: """🛑 *BUILD ABORTED*

            🔧 *Job:* ${env.JOB_NAME}
            🔢 *Build:* #${env.BUILD_NUMBER}
            🌍 *Environment:* ${params.environment}

            ⏹️ Pipeline execution was stopped.
            📋 <${env.BUILD_URL}console|View Console Logs>""",
                        failOnError: false
                )
            }

    cleanup {
            cleanWs()
            }
        }
}
