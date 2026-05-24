pipeline {
    agent { label 'linux' }
    tools {
        maven 'Maven'
    }
    parameters {
        choice(name: 'ENVIRONMENT', choices: ['dev', 'qa', 'prod'], description: 'Deploy target')
        booleanParam(name: 'SKIP_TESTS', defaultValue: false, description: 'Skip tests?')
    }
    stages {
        stage('Checkout') {
            steps {
                echo "Building on: ${env.NODE_NAME}"
                echo "Target environment: ${params.ENVIRONMENT}"
                checkout scm
            }
        }
        stage('Build') {
            steps {
                sh 'mvn clean compile -DskipTests'
                echo 'Compilation complete'
            }
        }
        stage('Tests') {
            when {
                expression { return !params.SKIP_TESTS }
            }
            steps {
                echo 'Running tests...'
                sh 'mvn test'
            }
        }
        stage('Package') {
            steps {
                sh 'mvn package -DskipTests'
                echo "Artifact: target/hello-app-1.0.0.war"
            }
        }
        stage('Deploy') {
            when {
                expression { params.ENVIRONMENT == 'prod' }
            }
            steps {
                input message: 'Deploy to PRODUCTION?', ok: 'Yes, deploy'
                echo 'Deploying to PRODUCTION...'
            }
        }
    }
    post {
        success { echo "SUCCESS: ${params.ENVIRONMENT}" }
        failure { echo "FAILED: Check console output" }
        always  { echo "Build #${env.BUILD_NUMBER} finished" }
    }
}
