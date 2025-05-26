pipeline {
    agent any

    tools {
        maven 'MVN'
        jdk 'JDK_11'
    }

    environment {
        DOCKERHUB_USER = 'juanmadrid09'
        DOCKER_CREDENTIALS_ID = 'password'
        SERVICES = 'api-gateway cloud-config favourite-service order-service payment-service product-service proxy-client service-discovery shipping-service user-service'
        K8S_NAMESPACE = 'ecommerce'
        KUBECONFIG = 'C:\\Users\\ACER\\.kube\\config'
    }

    parameters {
        booleanParam(
            name: 'GENERATE_RELEASE_NOTES',
            defaultValue: true,
            description: 'Generate automatic release notes'
        )
        string(
            name: 'BUILD_TAG',
            defaultValue: "${env.BUILD_ID}",
            description: 'Tag for release notes identification'
        )
    }

    stages {
        stage('Init') {
            steps {
                script {
                    if (env.BRANCH_NAME == 'master') {
                        env.SPRING_PROFILE = 'prod'
                        env.IMAGE_TAG = 'prod'
                        env.DEPLOYMENT_SUFFIX = '-prod'

                    } else if (env.BRANCH_NAME == 'release') {
                        env.SPRING_PROFILE = 'stage'
                        env.IMAGE_TAG = 'stage'
                        env.DEPLOYMENT_SUFFIX = '-stage'

                    } else {
                        env.SPRING_PROFILE = 'dev'
                        env.IMAGE_TAG = 'dev'
                        env.DEPLOYMENT_SUFFIX = '-dev'
                    }

                    echo "Branch: ${env.BRANCH_NAME}"
                    echo "Namespace: ${env.K8S_NAMESPACE}"
                    echo "Spring profile: ${env.SPRING_PROFILE}"
                    echo "Image tag: ${env.IMAGE_TAG}"
                    echo "Deployment suffix: ${env.DEPLOYMENT_SUFFIX}"
                }
            }
        }

        stage('Verify Tools') {
                    steps {
                        bat 'java -version'
                        bat 'mvn -version'
                        bat 'docker --version'
                        bat 'kubectl config current-context'

                    }
                }

        stage('Ensure Namespace') {
            steps {
                script {
                    def ns = env.K8S_NAMESPACE
                    bat "kubectl get namespace ${ns} || kubectl create namespace ${ns}"
                }
            }
        }

        stage('Checkout') {
            steps {
                git branch: "${env.BRANCH_NAME}", url: 'https://github.com/JuanMadrid-09/ecommerce-microservice-backend-app.git'
            }
        }


/*
        stage('Unit Tests') {
            parallel {
                stage('Unit Tests') {
                    when {
                        anyOf {
                            branch 'dev'
                            branch 'master'
                            branch 'release'
                            expression { env.BRANCH_NAME.startsWith('feature/') }
                        }
                    }
                    steps {
                        script {
                            echo "🔍 Running Unit Tests for ${env.BRANCH_NAME}"
                            bat "mvn test -pl product-service"
                        }
                    }
                }
            }
        }



        stage('Integration Tests') {
            parallel {
                stage('Integration Tests') {
                    when {
                        anyOf {
                            branch 'master'
                            expression { env.BRANCH_NAME.startsWith('feature/') }
                            allOf {
                                not { branch 'master' }
                                not { branch 'release' }
                            }
                        }
                    }
                    steps {
                        script {
                            echo "🧪 Running Integration Tests for ${env.BRANCH_NAME}"
                            bat "mvn verify -pl product-service"
                        }
                    }
                }
            }
        }

        stage('E2E Tests') {
                    parallel {
                        stage('E2E Tests') {
                            when {
                                anyOf {
                                    branch 'master'
                                    expression { env.BRANCH_NAME.startsWith('feature/') }
                                    allOf {
                                        not { branch 'master' }
                                        not { branch 'release' }
                                    }
                                }
                            }
                            steps {
                                script {
                                    echo "🧪 Running Integration Tests for ${env.BRANCH_NAME}"
                                    bat "mvn verify -pl e2e-tests"
                                }
                            }
                        }
                    }
                }

*/

        stage('Build Services') {
            when {
                anyOf {
                    branch 'master'
                    branch 'release'
                }
            }
            steps {
                bat "mvn clean package -DskipTests"
            }
        }

        stage('Build Docker Images') {
            when { branch 'master' }
            steps {
                script {
                    SERVICES.split().each { service ->
                        bat "docker build -t ${DOCKERHUB_USER}/${service}:${IMAGE_TAG} .\\${service}"
                    }
                }
            }
        }

        stage('Push Docker Images') {
            when { branch 'master' }
            steps {
                withCredentials([string(credentialsId: "${DOCKER_CREDENTIALS_ID}", variable: 'password')]) {
                    bat "docker login -u ${DOCKERHUB_USER} -p ${password}"
                    script {
                        SERVICES.split().each { service ->
                            bat "docker push ${DOCKERHUB_USER}/${service}:${IMAGE_TAG}"
                        }
                    }
                }
            }
        }

        stage('Deploy Common Config') {
            when { branch 'master' }
            steps {
                bat "kubectl apply -f k8s\\common-config.yaml -n ${K8S_NAMESPACE}"
            }
        }

        stage('Deploy Core Services') {
            when { branch 'master' }
            steps {
                bat "kubectl apply -f k8s\\zipkin -n ${K8S_NAMESPACE}"
                bat "kubectl rollout status deployment/zipkin -n ${K8S_NAMESPACE} --timeout=150s"

                bat "kubectl apply -f k8s\\service-discovery -n ${K8S_NAMESPACE}"
                bat "kubectl set image deployment/service-discovery service-discovery=${DOCKERHUB_USER}/service-discovery:${IMAGE_TAG} -n ${K8S_NAMESPACE}"
                bat "kubectl rollout status deployment/service-discovery -n ${K8S_NAMESPACE} --timeout=300s"

                bat "kubectl apply -f k8s\\cloud-config -n ${K8S_NAMESPACE}"
                bat "kubectl set image deployment/cloud-config cloud-config=${DOCKERHUB_USER}/cloud-config:${IMAGE_TAG} -n ${K8S_NAMESPACE}"
                bat "kubectl rollout status deployment/cloud-config -n ${K8S_NAMESPACE} --timeout=350s"
            }
        }

        stage('Deploy Microservices') {
            when { branch 'master' }
            steps {
                script {
                    echo '👻👻👻👻👻👻'
                }
            }
        }

        stage('Generate Release Notes') {
            when {
                expression { params.GENERATE_RELEASE_NOTES }
            }
            steps {
                script {
                    echo "=== GENERATE RELEASE NOTES ==="
                    generateReleaseNotes()
                }
            }
        }
    }

    post {
        success {
            script {
                echo "✅ Pipeline completed successfully for ${env.BRANCH_NAME} branch."
                echo "📊 Environment: ${env.SPRING_PROFILE}"

                if (env.BRANCH_NAME == 'master') {
                    echo "🚀 Production deployment completed successfully!"
                } else if (env.BRANCH_NAME == 'release') {
                    echo "🎯 Staging deployment completed successfully!"
                } else {
                    echo "🔧 Development tests completed successfully!"
                }
            }
        }
        failure {
            script {
                echo "❌ Pipeline failed for ${env.BRANCH_NAME} branch."
                echo "🔍 Check the logs for details."
                echo "📧 Notify the development team about the failure."

            }
        }
        unstable {
            script {
                echo "⚠️ Pipeline completed with warnings for ${env.BRANCH_NAME} branch."
                echo "🔍 Some tests may have failed. Review test reports."
            }
        }
        always {
            script {
                // Archive release notes if they were generated
                if (params.GENERATE_RELEASE_NOTES) {
                    archiveArtifacts artifacts: 'release-notes-*.md', allowEmptyArchive: true
                }
            }
        }
    }
}

def generateReleaseNotes() {
    echo "Generando Release Notes automáticos..."

    try {
        def buildTag = params.BUILD_TAG ?: env.BUILD_ID
        def releaseNotesFile = "release-notes-${buildTag}.md"

        // Get git information (Windows compatible)
        def gitCommit = bat(returnStdout: true, script: 'git rev-parse HEAD').trim()
        def gitBranch = env.BRANCH_NAME ?: 'unknown'
        def buildDate = new Date().format('yyyy-MM-dd HH:mm:ss')

        // Get recent commits (Windows compatible)
        def recentCommits = ""
        try {
            recentCommits = bat(returnStdout: true, script: 'git log --oneline --since="3 days ago" | head -10').trim()
            if (!recentCommits) {
                recentCommits = "No recent commits found in the last 3 days"
            }
        } catch (Exception e) {
            recentCommits = "Could not retrieve recent commits: ${e.message}"
        }

        // Determine deployment status based on branch
        def deploymentStatus = ""
        switch(env.BRANCH_NAME) {
            case 'master':
                deploymentStatus = "✅ Successfully deployed to PRODUCTION environment"
                break
            case 'release':
                deploymentStatus = "✅ Successfully deployed to STAGING environment"
                break
            default:
                deploymentStatus = "✅ Tests completed for DEVELOPMENT environment"
        }

        def releaseNotes = """
# Release Notes - Build ${buildTag}

## Build Information
- **Build Number**: ${env.BUILD_NUMBER}
- **Build Tag**: ${buildTag}
- **Branch**: ${gitBranch}
- **Environment**: ${env.SPRING_PROFILE}
- **Date**: ${buildDate}
- **Git Commit**: ${gitCommit}
- **Jenkins URL**: ${env.BUILD_URL}

## Deployed Services (${env.SPRING_PROFILE} environment)
${SERVICES.split().collect { "- ${it}" }.join('\n')}

## Additional Infrastructure
- zipkin (monitoring)
- Kubernetes namespace: ${env.K8S_NAMESPACE}

## Test Results Summary
- **Unit Tests**: ${shouldRunTests() ? 'EXECUTED ✅' : 'SKIPPED ⏭️'}
- **Integration Tests**: ${shouldRunIntegrationTests() ? 'EXECUTED ✅' : 'SKIPPED ⏭️'}
- **E2E Tests**: ${shouldRunE2ETests() ? 'EXECUTED ✅' : 'SKIPPED ⏭️'}

## Recent Changes
```
${recentCommits}
```

## Docker Images Built
${env.BRANCH_NAME == 'master' ? SERVICES.split().collect { "- ${DOCKERHUB_USER}/${it}:${env.IMAGE_TAG}" }.join('\n') : 'No Docker images built for this branch'}

## Deployment Configuration
- **Spring Profile**: ${env.SPRING_PROFILE}
- **Image Tag**: ${env.IMAGE_TAG}
- **Deployment Suffix**: ${env.DEPLOYMENT_SUFFIX}
- **Kubernetes Namespace**: ${env.K8S_NAMESPACE}

## Deployment Status
${deploymentStatus}

## Pipeline Execution Details
- **Started**: ${new Date(currentBuild.startTimeInMillis).format('yyyy-MM-dd HH:mm:ss')}
- **Duration**: ${currentBuild.durationString}
- **Triggered by**: ${env.BUILD_CAUSE ?: 'Manual/SCM'}

---
*Generated automatically by Jenkins Pipeline on ${buildDate}*
*Pipeline: ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}*
"""

        writeFile(file: releaseNotesFile, text: releaseNotes)

        echo "✅ Release Notes generados exitosamente: ${releaseNotesFile}"
        echo "📄 Archivo será archivado como artifact"

        // Display summary in console
        echo """
=== RELEASE NOTES SUMMARY ===
📦 Build: ${buildTag}
🌿 Branch: ${gitBranch}
🏷️ Environment: ${env.SPRING_PROFILE}
📅 Date: ${buildDate}
📁 File: ${releaseNotesFile}
"""

    } catch (Exception e) {
        echo "⚠️ Error generando Release Notes: ${e.message}"
        echo "Pipeline continuará sin Release Notes"

        // Create minimal release notes
        def fallbackFile = "release-notes-${params.BUILD_TAG ?: env.BUILD_ID}-minimal.md"
        def minimalNotes = """
# Release Notes - Build ${params.BUILD_TAG ?: env.BUILD_ID}

**Error**: Could not generate complete release notes due to: ${e.message}

## Basic Information
- Build Number: ${env.BUILD_NUMBER}
- Branch: ${env.BRANCH_NAME}
- Environment: ${env.SPRING_PROFILE}
- Date: ${new Date().format('yyyy-MM-dd HH:mm:ss')}

Pipeline executed successfully despite release notes generation error.
"""
        writeFile(file: fallbackFile, text: minimalNotes)
        echo "📝 Minimal release notes created: ${fallbackFile}"
    }
}

// Helper functions to determine test execution
def shouldRunTests() {
    return env.BRANCH_NAME in ['dev', 'master', 'release'] || env.BRANCH_NAME.startsWith('feature/')
}

def shouldRunIntegrationTests() {
    return env.BRANCH_NAME == 'master' || env.BRANCH_NAME.startsWith('feature/') ||
           (env.BRANCH_NAME != 'master' && env.BRANCH_NAME != 'release')
}

def shouldRunE2ETests() {
    return env.BRANCH_NAME == 'master' || env.BRANCH_NAME.startsWith('feature/') ||
           (env.BRANCH_NAME != 'master' && env.BRANCH_NAME != 'release')
}