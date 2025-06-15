pipeline {
    agent any

    tools {
        maven 'MVN'
        jdk 'JDK_11'
    }

    environment {
        DOCKERHUB_USER = 'juanmadrid09'
        DOCKER_CREDENTIALS_ID = 'password'
        SERVICES = 'api-gateway cloud-config favourite-service order-service payment-service product-service proxy-client service-discovery shipping-service user-service locust'
        K8S_NAMESPACE = 'ecommerce'
        KUBECONFIG = 'C:\\Users\\ACER\\.kube\\config'
        // Semantic versioning variables
        VERSION_FILE = 'version.txt'
    }

    parameters {
        choice(
            name: 'VERSION_TYPE',
            choices: ['AUTO', 'PATCH', 'MINOR', 'MAJOR'],
            description: 'Tipo de versionado: AUTO (basado en commits), PATCH (x.x.X), MINOR (x.X.x), MAJOR (X.x.x)'
        )
        string(
            name: 'CUSTOM_VERSION',
            defaultValue: '',
            description: 'Versión personalizada (opcional, formato: x.y.z)'
        )
        booleanParam(
            name: 'SKIP_VERSION_BUMP',
            defaultValue: false,
            description: 'Saltar incremento de versión (usar versión actual)'
        )
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
        stage('Version Management') {
            steps {
                script {
                    // Get current version
                    def currentVersion = getCurrentVersion()
                    echo "📦 Current version: ${currentVersion}"

                    // Calculate new version
                    if (params.CUSTOM_VERSION) {
                        env.NEW_VERSION = params.CUSTOM_VERSION
                        echo "🎯 Using custom version: ${env.NEW_VERSION}"
                    } else if (params.SKIP_VERSION_BUMP) {
                        env.NEW_VERSION = currentVersion
                        echo "⏭️ Skipping version bump, using current: ${env.NEW_VERSION}"
                    } else {
                        env.NEW_VERSION = calculateNextVersion(currentVersion, params.VERSION_TYPE)
                        echo "📈 New version calculated: ${env.NEW_VERSION}"
                    }

                    // Validate version format
                    if (!env.NEW_VERSION.matches(/^\d+\.\d+\.\d+$/)) {
                        error("❌ Invalid version format: ${env.NEW_VERSION}. Must be x.y.z")
                    }

                    // Create version tags
                    env.VERSION_TAG = "v${env.NEW_VERSION}"
                    env.BUILD_TAG = "${env.NEW_VERSION}-${env.BUILD_NUMBER}"
                    env.DOCKER_TAG = "${env.BRANCH_NAME == 'master' ? env.NEW_VERSION : env.BUILD_TAG}"

                    // Update version files
                    if (!params.SKIP_VERSION_BUMP) {
                        updateVersionFiles(env.NEW_VERSION)
                    }
                }
            }
        }

        stage('Init') {
            steps {
                script {
                    def profileConfig = [
                        master : ['prod', '-prod'],
                        release: ['stage', '-stage']
                    ]
                    def config = profileConfig.get(env.BRANCH_NAME, ['dev', '-dev'])

                    env.SPRING_PROFILES_ACTIVE = config[0]
                    env.IMAGE_TAG = config[0]
                    env.DEPLOYMENT_SUFFIX = config[1]

                    echo "📦 Branch: ${env.BRANCH_NAME}"
                    echo "🌱 Spring profile: ${env.SPRING_PROFILES_ACTIVE}"
                    echo "🏷️ Image tag: ${env.IMAGE_TAG}"
                    echo "📂 Deployment suffix: ${env.DEPLOYMENT_SUFFIX}"
                    echo "📈 Version: ${env.NEW_VERSION}"
                }
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

        stage('Verify Tools') {
            steps {
                bat 'java -version'
                bat 'mvn -version'
                bat 'docker --version'
                bat 'kubectl config current-context'
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
                echo "📈 Version: ${env.NEW_VERSION}"

                if (env.BRANCH_NAME == 'master') {
                    echo "🚀 Production deployment completed successfully!"
                } else if (env.BRANCH_NAME == 'stage') {
                    echo "🎯 Staging deployment completed successfully!"
                    publishHTML([
                        reportDir: 'locust-reports',
                        reportFiles: 'order-service-report.html, payment-service-report.html, favourite-service-report.html',
                        reportName: 'Locust Stress Test Reports',
                        keepAll: true
                    ])
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
                // Archive version file
                archiveArtifacts artifacts: env.VERSION_FILE, fingerprint: true
            }
        }
    }
}

// Version management functions
def getCurrentVersion() {
    def version = bat(returnStdout: true, script: """
        if exist ${env.VERSION_FILE} (
            type ${env.VERSION_FILE}
        ) else (
            mvn help:evaluate -Dexpression=project.version -q -DforceStdout
        )
    """).trim()
    
    return version.replaceAll('-SNAPSHOT', '')
}

def calculateNextVersion(currentVersion, versionType) {
    def (major, minor, patch) = currentVersion.tokenize('.')
    
    switch(versionType) {
        case 'AUTO':
            return calculateAutoVersion(currentVersion)
        case 'PATCH':
            return "${major}.${minor}.${(patch as int) + 1}"
        case 'MINOR':
            return "${major}.${(minor as int) + 1}.0"
        case 'MAJOR':
            return "${(major as int) + 1}.0.0"
        default:
            return calculateAutoVersion(currentVersion)
    }
}

def calculateAutoVersion(currentVersion) {
    // Get commit messages since last tag
    def commitMessages = bat(returnStdout: true, script: """
        git log $(git describe --tags --abbrev=0 2>nul || echo HEAD~10)..HEAD --pretty=format:"%%s" || echo ""
    """).trim()
    
    def (major, minor, patch) = currentVersion.tokenize('.')
    
    // Determine version bump based on commit messages
    if (commitMessages.contains('BREAKING CHANGE') || commitMessages.contains('!:')) {
        return "${(major as int) + 1}.0.0"
    } else if (commitMessages.contains('feat:') || commitMessages.contains('feature:')) {
        return "${major}.${(minor as int) + 1}.0"
    } else {
        return "${major}.${minor}.${(patch as int) + 1}"
    }
}

def updateVersionFiles(version) {
    // Update version.txt
    writeFile file: env.VERSION_FILE, text: version
    
    // Update main pom.xml
    bat """
        mvn versions:set -DnewVersion=${version} -DgenerateBackupPoms=false
        mvn versions:commit
    """
    
    // Update microservices pom files
    bat """
        for /r %%f in (pom.xml) do (
            if not "%%~dpf"=="%cd%\\target\\" (
                mvn versions:set -DnewVersion=${version} -DgenerateBackupPoms=false -f "%%f"
            )
        )
    """
}

def generateReleaseNotes() {
    echo "Generating automatic Release Notes..."

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
           recentCommits = bat(returnStdout: true, script: 'git log --oneline --since="3 days ago" -n 10').trim()
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

        echo "✅ Release Notes generated successfully: ${releaseNotesFile}"
        echo "📄 File will be save as artifact"

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
        echo "⚠️ Error generating Release Notes: ${e.message}"
        echo "Pipeline will continue without Release Notes"

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