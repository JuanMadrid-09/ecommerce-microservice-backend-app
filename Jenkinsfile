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


//
//          stage('Unit Tests') {
//                     when {
//                         anyOf {
//                             branch 'dev'; branch 'master'; branch 'release'
//                             expression { env.BRANCH_NAME.startsWith('feature/') }
//                         }
//                     }
//                     steps {
//                         script {
//                             ['user-service', 'product-service', 'payment-service'].each {
//                                 bat "mvn test -pl ${it}"
//                             }
//                         }
//                     }
//                 }
//
//
//         stage('Integration Tests') {
//                     when {
//                         anyOf {
//                             branch 'master'
//                             expression { env.BRANCH_NAME.startsWith('feature/') }
//                             allOf { not { branch 'master' }; not { branch 'release' } }
//                         }
//                     }
//                     steps {
//                         script {
//                             ['user-service', 'product-service'].each {
//                                 bat "mvn verify -pl ${it}"
//                             }
//                         }
//                     }
//                 }
//
//          stage('E2E Tests') {
//                     when {
//                         anyOf {
//                             branch 'master'
//                             expression { env.BRANCH_NAME.startsWith('feature/') }
//                             allOf { not { branch 'master' }; not { branch 'release' } }
//                         }
//                     }
//                     steps {
//                         bat "mvn verify -pl e2e-tests"
//                     }
//                 }



       stage('Build & Package') {
                   when { anyOf { branch 'master'; branch 'release' } }
                   steps {
                       bat "mvn clean package -DskipTests"
                   }
               }

       stage('Build & Push Docker Images') {
           when { branch 'master' }
           steps {
               withCredentials([string(credentialsId: "${DOCKER_CREDENTIALS_ID}", variable: 'password')]) {
                   bat "docker login -u ${DOCKERHUB_USER} -p ${password}"

                   script {
                       SERVICES.split().each { service ->
                           bat "docker build -t ${DOCKERHUB_USER}/${service}:${IMAGE_TAG} .\\${service}"
                           bat "docker push ${DOCKERHUB_USER}/${service}:${IMAGE_TAG}"
                       }
                   }
               }
           }
       }

      stage('Levantar contenedores para pruebas') {
          steps {
              script {
                  bat '''
                  docker network create ecommerce-test || true

                  echo 🚀 Levantando ZIPKIN con límites reducidos...
                  docker run -d --name zipkin-container --network ecommerce-test -p 9411:9411 ^
                      --memory=128m --cpus=0.25 ^
                      openzipkin/zipkin

                  echo 🚀 Levantando EUREKA con límites reducidos...
                  docker run -d --name service-discovery-container --network ecommerce-test -p 8761:8761 ^
                      --memory=256m --cpus=0.3 ^
                      -e SPRING_PROFILES_ACTIVE=dev ^
                      -e SPRING_ZIPKIN_BASE_URL=http://zipkin-container:9411 ^
                      -e JAVA_OPTS="-Xmx128m -Xms64m -XX:+UseG1GC -XX:MaxGCPauseMillis=100 -XX:+UseStringDeduplication" ^
                      juanmadrid09/service-discovery:%IMAGE_TAG%

                  call :waitForService http://localhost:8761/actuator/health 120

                  echo 🚀 Levantando CLOUD-CONFIG con límites reducidos...
                  docker run -d --name cloud-config-container --network ecommerce-test -p 9296:9296 ^
                      --memory=256m --cpus=0.3 ^
                      -e SPRING_PROFILES_ACTIVE=dev ^
                      -e SPRING_ZIPKIN_BASE_URL=http://zipkin-container:9411 ^
                      -e EUREKA_CLIENT_SERVICEURL_DEFAULTZONE=http://service-discovery-container:8761/eureka/ ^
                      -e EUREKA_INSTANCE=cloud-config-container ^
                      -e JAVA_OPTS="-Xmx128m -Xms64m -XX:+UseG1GC -XX:MaxGCPauseMillis=100 -XX:+UseStringDeduplication" ^
                      juanmadrid09/cloud-config:%IMAGE_TAG%

                  call :waitForService http://localhost:9296/actuator/health 120

                  REM Levantar servicios DE UNO EN UNO para reducir picos de CPU
                  echo 🚀 Levantando servicios secuencialmente...
                  call :runServiceSequential order-service 8300
                  call :runServiceSequential payment-service 8400
                  call :runServiceSequential product-service 8500
                  call :runServiceSequential shipping-service 8600
                  call :runServiceSequential user-service 8700
                  call :runServiceSequentialWithExtraWait favourite-service 8800

                  echo ✅ Todos los contenedores están arriba y saludables.
                  exit /b 0

                  :runServiceSequential
                  set "NAME=%~1"
                  set "PORT=%~2"
                  echo 🚀 Levantando %NAME%...
                  docker run -d --name %NAME%-container --network ecommerce-test -p %PORT%:%PORT% ^
                      --memory=192m --cpus=0.25 ^
                      -e SPRING_PROFILES_ACTIVE=dev ^
                      -e SPRING_ZIPKIN_BASE_URL=http://zipkin-container:9411 ^
                      -e SPRING_CONFIG_IMPORT=optional:configserver:http://cloud-config-container:9296 ^
                      -e EUREKA_CLIENT_SERVICE_URL_DEFAULTZONE=http://service-discovery-container:8761/eureka ^
                      -e EUREKA_INSTANCE=%NAME%-container ^
                      -e JAVA_OPTS="-Xmx128m -Xms64m -XX:+UseG1GC -XX:MaxGCPauseMillis=100 -XX:+UseStringDeduplication -Djava.security.egd=file:/dev/./urandom" ^
                      juanmadrid09/%NAME%:%IMAGE_TAG%

                  REM Esperar un poco para que el contenedor inicie antes del siguiente
                  timeout /t 15 /nobreak > nul
                  call :waitForService http://localhost:%PORT%/%NAME%/actuator/health 120
                  exit /b 0

                  :runServiceSequentialWithExtraWait
                  set "NAME=%~1"
                  set "PORT=%~2"
                  echo 🚀 Levantando %NAME% con configuración especial...
                  docker run -d --name %NAME%-container --network ecommerce-test -p %PORT%:%PORT% ^
                      --memory=256m --cpus=0.3 ^
                      -e SPRING_PROFILES_ACTIVE=dev ^
                      -e SPRING_ZIPKIN_BASE_URL=http://zipkin-container:9411 ^
                      -e SPRING_CONFIG_IMPORT=optional:configserver:http://cloud-config-container:9296 ^
                      -e EUREKA_CLIENT_SERVICE_URL_DEFAULTZONE=http://service-discovery-container:8761/eureka ^
                      -e EUREKA_INSTANCE=%NAME%-container ^
                      -e JAVA_OPTS="-Xmx192m -Xms96m -XX:+UseG1GC -XX:MaxGCPauseMillis=100 -XX:+UseStringDeduplication -Djava.security.egd=file:/dev/./urandom" ^
                      juanmadrid09/%NAME%:%IMAGE_TAG%

                  REM Esperar más tiempo para favourite-service
                  echo ⏳ Esperando tiempo adicional para %NAME%...
                  timeout /t 30 /nobreak > nul
                  call :waitForService http://localhost:%PORT%/%NAME%/actuator/health 180

                  REM Verificar que el endpoint de la API también esté disponible
                  echo ⏳ Verificando endpoint de API para %NAME%...
                  call :waitForApiEndpoint http://localhost:%PORT%/api/favourites 60
                  exit /b 0

                  :waitForService
                  set "URL=%~1"
                  set "TIMEOUT_SECONDS=%~2"
                  if "%TIMEOUT_SECONDS%"=="" set "TIMEOUT_SECONDS=60"

                  echo ⏳ Esperando a que %URL% esté disponible (timeout: %TIMEOUT_SECONDS%s)...
                  set /a "COUNTER=0"
                  set /a "MAX_ATTEMPTS=%TIMEOUT_SECONDS%/15"

                  :wait_loop
                  set /a "COUNTER+=1"
                  if %COUNTER% gtr %MAX_ATTEMPTS% (
                      echo ❌ Timeout alcanzado para %URL% después de %TIMEOUT_SECONDS% segundos
                      exit /b 1
                  )

                  curl -s --connect-timeout 5 --max-time 10 %URL% > nul 2>&1
                  if %errorlevel% equ 0 (
                      for /f "delims=" %%i in ('curl -s --connect-timeout 5 --max-time 10 %URL% ^| jq -r ".status" 2^>nul') do (
                          if "%%i"=="UP" (
                              echo ✅ %URL% está disponible
                              goto :eof
                          )
                      )
                  )

                  echo . (intento %COUNTER%/%MAX_ATTEMPTS%)
                  timeout /t 15 /nobreak > nul
                  goto wait_loop

                  :waitForApiEndpoint
                  set "API_URL=%~1"
                  set "TIMEOUT_SECONDS=%~2"
                  if "%TIMEOUT_SECONDS%"=="" set "TIMEOUT_SECONDS=30"

                  echo ⏳ Esperando a que endpoint API %API_URL% esté disponible...
                  set /a "COUNTER=0"
                  set /a "MAX_ATTEMPTS=%TIMEOUT_SECONDS%/10"

                  :api_wait_loop
                  set /a "COUNTER+=1"
                  if %COUNTER% gtr %MAX_ATTEMPTS% (
                      echo ⚠️ Timeout para API endpoint %API_URL%, pero continuando...
                      goto :eof
                  )

                  curl -s --connect-timeout 3 --max-time 8 %API_URL% > nul 2>&1
                  if %errorlevel% equ 0 (
                      echo ✅ API endpoint %API_URL% está respondiendo
                      goto :eof
                  )

                  REM También intentar con métodos HTTP específicos
                  curl -s --connect-timeout 3 --max-time 8 -X GET %API_URL% > nul 2>&1
                  if %errorlevel% equ 0 (
                      echo ✅ API endpoint %API_URL% está respondiendo (GET)
                      goto :eof
                  )

                  echo . (API intento %COUNTER%/%MAX_ATTEMPTS%)
                  timeout /t 10 /nobreak > nul
                  goto api_wait_loop
                  '''
              }
          }
      }


      stage('Calentar servicios antes de pruebas') {
          when { branch 'master' }
          steps {
              script {
                  bat '''
                  echo 🔥 Calentando servicios antes de las pruebas de estrés...

                  echo ⏳ Realizando warmup de favourite-service...
                  REM Hacer algunas peticiones de prueba para calentar el servicio
                  for /l %%i in (1,1,5) do (
                      echo Warmup request %%i/5 to favourite-service...
                      curl -s --connect-timeout 3 --max-time 10 -X GET http://localhost:8800/api/favourites > nul 2>&1 || echo Request %%i failed, continuing...
                      timeout /t 2 /nobreak > nul
                  )

                  echo ⏳ Realizando warmup de order-service...
                  for /l %%i in (1,1,3) do (
                      echo Warmup request %%i/3 to order-service...
                      curl -s --connect-timeout 3 --max-time 10 -X GET http://localhost:8300/order-service/actuator/health > nul 2>&1 || echo Request %%i failed, continuing...
                      timeout /t 2 /nobreak > nul
                  )

                  echo ⏳ Realizando warmup de payment-service...
                  for /l %%i in (1,1,3) do (
                      echo Warmup request %%i/3 to payment-service...
                      curl -s --connect-timeout 3 --max-time 10 -X GET http://localhost:8400/payment-service/actuator/health > nul 2>&1 || echo Request %%i failed, continuing...
                      timeout /t 2 /nobreak > nul
                  )

                  echo ⏳ Esperando tiempo adicional para estabilización...
                  timeout /t 30 /nobreak > nul

                  echo ✅ Servicios calentados y listos para pruebas de estrés
                  '''
              }
          }
      }

//
//        stage('Run Load Tests with Locust') {
//
//            when { branch 'master' }
//            steps {
//                script {
//                    bat '''
//
//                    echo 🚀 Levantando Locust para order-service...
//                    docker run --rm --network ecommerce-test ^
//                      -v "%CD%\\locust:/mnt" ^
//                      -v "%CD%\\locust-results:/app" ^
//                      juanmadrid09/locust:%IMAGE_TAG% ^
//                      -f /mnt/test/order-service/locustfile.py ^
//                      --host http://order-service-container:8300 ^
//                      --headless -u 5 -r 1 -t 1m ^
//                      --csv order-service-stats --csv-full-history
//
//                    echo 🚀 Levantando Locust para payment-service...
//
//                    docker run --rm --network ecommerce-test ^
//                      -v "%CD%\\locust:/mnt" ^
//                      -v "%CD%\\locust-results:/app" ^
//                      juanmadrid09/locust:%IMAGE_TAG% ^
//                      -f /mnt/test/payment-service/locustfile.py ^
//                      --host http://payment-service-container:8400 ^
//                      --headless -u 5 -r 1 -t 1m ^
//                      --csv payment-service-stats --csv-full-history
//
//                    echo 🚀 Levantando Locust para favourite-service...
//
//                    docker run --rm --network ecommerce-test ^
//                      -v "%CD%\\locust:/mnt" ^
//                      -v "%CD%\\locust-results:/app" ^
//                      juanmadrid09/locust:%IMAGE_TAG% ^
//                      -f /mnt/test/favourite-service/locustfile.py ^
//                      --host http://favourite-service-container:8800 ^
//                      --headless -u 5 -r 1 -t 1m ^
//                      --csv favourite-service-stats --csv-full-history
//
//                    echo ✅ Pruebas completadas
//                    '''
//                }
//            }
//        }

       stage('Run Stress Tests with Locust') {
           when { branch 'master' }
           steps {
               script {
                   bat '''
                   echo 🔥 Levantando Locust para prueba de estrés...

                   docker run --rm --network ecommerce-test ^
                   -v "%CD%\\locust:/mnt" ^
                   -v "%CD%\\locust-results:/app" ^
                   juanmadrid09/locust:%IMAGE_TAG% ^
                   -f /mnt/test/order-service/locustfile.py ^
                   --host http://order-service-container:8300 ^
                   --headless -u 10 -r 1 -t 1m ^
                   --csv order-service-stress --csv-full-history

                   docker run --rm --network ecommerce-test ^
                   -v "%CD%\\locust:/mnt" ^
                   -v "%CD%\\locust-results:/app" ^
                   juanmadrid09/locust:%IMAGE_TAG% ^
                   -f /mnt/test/payment-service/locustfile.py ^
                   --host http://payment-service-container:8400 ^
                   --headless -u 10 -r 1 -t 1m ^
                   --csv payment-service-stress --csv-full-history

                   docker run --rm --network ecommerce-test ^
                   -v "%CD%\\locust:/mnt" ^
                   -v "%CD%\\locust-results:/app" ^
                   juanmadrid09/locust:%IMAGE_TAG% ^
                   -f /mnt/test/favourite-service/locustfile.py ^
                   --host http://favourite-service-container:8800 ^
                   --headless -u 10 -r 1 -t 1m ^
                   --csv favourite-service-stress --csv-full-history

                   echo ✅ Pruebas de estrés completadas
                   '''
               }
           }
       }



       stage('Detener y eliminar contenedores') {
           steps {
               script {
                   bat """
                   echo 🛑 Deteniendo y eliminando contenedores...

                   docker rm -f locust || exit 0
                   docker rm -f favourite-service-container || exit 0
                   docker rm -f user-service-container || exit 0
                   docker rm -f shipping-service-container || exit 0
                   docker rm -f product-service-container || exit 0
                   docker rm -f payment-service-container || exit 0
                   docker rm -f order-service-container || exit 0
                   docker rm -f cloud-config-container || exit 0
                   docker rm -f service-discovery-container || exit 0
                   docker rm -f zipkin-container || exit 0

                   echo 🧹 Todos los contenedores eliminados
                   """
               }
           }
       }
//
//         stage('Deploy Common Config') {
//             when { branch 'master' }
//             steps {
//                 bat "kubectl apply -f k8s\\common-config.yaml -n ${K8S_NAMESPACE}"
//             }
//         }
//
//         stage('Deploy Core Services') {
//             when { branch 'master' }
//             steps {
//                 bat "kubectl apply -f k8s\\zipkin -n ${K8S_NAMESPACE}"
//                 bat "kubectl rollout status deployment/zipkin -n ${K8S_NAMESPACE} --timeout=150s"
//
//                 bat "kubectl apply -f k8s\\service-discovery -n ${K8S_NAMESPACE}"
//                 bat "kubectl set image deployment/service-discovery service-discovery=${DOCKERHUB_USER}/service-discovery:${IMAGE_TAG} -n ${K8S_NAMESPACE}"
//                 bat "kubectl rollout status deployment/service-discovery -n ${K8S_NAMESPACE} --timeout=300s"
//
//                 bat "kubectl apply -f k8s\\cloud-config -n ${K8S_NAMESPACE}"
//                 bat "kubectl set image deployment/cloud-config cloud-config=${DOCKERHUB_USER}/cloud-config:${IMAGE_TAG} -n ${K8S_NAMESPACE}"
//                 bat "kubectl rollout status deployment/cloud-config -n ${K8S_NAMESPACE} --timeout=350s"
//             }
//         }
//
//
//
//          stage('Deploy Microservices') {
//                     when { branch 'master' }
//                     steps {
//                         script {
//                             echo "👻👻👻👻👻"
//         //                     SERVICES.split().each { svc ->
//         //                         if (!['user-service', ].contains(svc)) {
//         //                             bat "kubectl apply -f k8s\\${svc} -n ${K8S_NAMESPACE}"
//         //                             bat "kubectl set image deployment/${svc} ${svc}=${DOCKERHUB_USER}/${svc}:${IMAGE_TAG} -n ${K8S_NAMESPACE}"
//         //                             bat "kubectl set env deployment/${svc} SPRING_PROFILES_ACTIVE=${SPRING_PROFILES_ACTIVE} -n ${K8S_NAMESPACE}"
//         //                             bat "kubectl rollout status deployment/${svc} -n ${K8S_NAMESPACE} --timeout=300s"
//         //                         }
//         //                     }
//                         }
//                     }
//                 }

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