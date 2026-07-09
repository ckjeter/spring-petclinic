// DevSecOps pipeline for spring-petclinic — 17-636 Mini Project
// Stages: Build+Test -> SonarQube -> Staging container -> ZAP baseline scan
//         -> publish ZAP report -> Ansible deploy to prod VM
//
// This file lives in the FORK (ckjeter/spring-petclinic) so that the
// pipeline definition itself is version-controlled with the app code
// ("pipeline as code"). A copy is kept in the mini-project repo for the
// deliverables zip.

pipeline {
    agent any

    triggers {
        // Poll SCM every ~2 minutes (assignment: "build triggers to poll SCM")
        pollSCM('H/2 * * * *')
    }

    options {
        timestamps()
    }

    environment {
        STAGING_NAME = 'petclinic-staging'
        NET = 'devsecops-net'
    }

    stages {

        stage('Build & Test') {
            steps {
                // Maven wrapper ships with petclinic; JDK17 comes from the
                // Jenkins image. -B = non-interactive batch mode.
                sh './mvnw -B clean verify'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                // 'sonarqube' is the server registered via JCasC; this wrapper
                // injects the URL + token so mvn sonar:sonar can reach it over
                // the docker network.
                withSonarQubeEnv('sonarqube') {
                    sh './mvnw -B org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=spring-petclinic'
                }
            }
        }

        stage('Stage App for ZAP') {
            steps {
                // Build a throwaway runtime image from the fresh jar and run
                // it on the shared network so ZAP can reach it by name.
                // Docker CLI talks to the HOST daemon (mounted socket).
                sh '''
                  cp target/spring-petclinic-*.jar petclinic.jar
                  docker rm -f ${STAGING_NAME} 2>/dev/null || true
                  docker build -t petclinic-staging -f Dockerfile.staging .
                  docker run -d --rm --name ${STAGING_NAME} --network ${NET} petclinic-staging
                  # wait until the app answers before scanning
                  for i in $(seq 1 30); do
                    docker run --rm --network ${NET} curlimages/curl:latest \
                      -sf http://${STAGING_NAME}:8080/ >/dev/null 2>&1 && break
                    sleep 2
                  done
                '''
            }
        }

        stage('ZAP Baseline Scan') {
            steps {
                // zap-baseline.py: passive scan by spidering the target.
                // -I = do not fail the build on WARN findings (report them).
                // Report is written into a named volume, then copied into the
                // workspace (DooD cannot mount workspace paths directly).
                sh '''
                  docker volume create zap_wrk >/dev/null
                  docker run --rm -v zap_wrk:/zap/wrk alpine \
                    sh -c 'chmod 777 /zap/wrk && rm -f /zap/wrk/zap-report.html /zap/wrk/zap.yaml'
                  docker run --rm --network ${NET} -v zap_wrk:/zap/wrk -w /zap/wrk \
                    ghcr.io/zaproxy/zaproxy:stable \
                    zap-baseline.py -t http://${STAGING_NAME}:8080 -r zap-report.html -I || true
                  docker run --rm -v zap_wrk:/zap/wrk alpine cat /zap/wrk/zap-report.html > zap-report.html
                  test -s zap-report.html
                '''
            }
            post {
                always {
                    // tear down staging whether the scan worked or not
                    sh 'docker rm -f ${STAGING_NAME} 2>/dev/null || true'
                    publishHTML(target: [
                        reportDir: '.',
                        reportFiles: 'zap-report.html',
                        reportName: 'ZAP Security Report',
                        keepAll: true,
                        alwaysLinkToLastBuild: true,
                        allowMissing: true
                    ])
                }
            }
        }

        stage('Deploy to Prod VM (Ansible)') {
            steps {
                // Same SSH remote-control pattern as Assignment 3, automated:
                // ansible runs inside this container, key + inventory are
                // mounted read-only at /ansible.
                sh '''
                  ansible-playbook -i /ansible/inventory.ini /ansible/deploy-petclinic.yml \
                    -e "jar_path=${WORKSPACE}/petclinic.jar"
                '''
            }
        }
    }
}
