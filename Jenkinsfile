def dockerRepo = "ghcr.io/icgc-argo/workflow-management"
def gitHubRepo = "icgc-argo/workflow-management"
def commit = "UNKNOWN"
def version = "UNKNOWN"

pipeline {
    agent {
        kubernetes {
            label 'wf-management'
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: jdk
    tty: true
    image: openjdk:11
    env:
      - name: DOCKER_HOST
        value: tcp://localhost:2375
  - name: dind-daemon
    image: docker:18.06-dind
    securityContext:
      privileged: true
      runAsUser: 0
    volumeMounts:
      - name: docker-graph-storage
        mountPath: /var/lib/docker
  - name: helm
    image: alpine/helm:2.12.3
    command:
    - cat
    tty: true
  - name: docker
    image: docker:18-git
    tty: true
    env:
      - name: DOCKER_HOST
        value: tcp://localhost:2375
      - name: HOME
        value: /home/jenkins/agent
  securityContext:
    runAsUser: 1000
  volumes:
  - name: docker-graph-storage
    emptyDir: {}
"""
        }
    }
    stages {
        stage('Prepare') {
            steps {
                script {
                    commit = sh(returnStdout: true, script: 'git describe --always').trim()
                }
                script {
                    version = readMavenPom().getVersion()
                }
            }
        }
        /* stage('Test') {
            steps {
                container('jdk') {
                    sh "./mvnw test"
                }
            }
        } */
        stage('Build Artifact & Publish') {
              when {
                  anyOf {
                     branch "nextflow_22-10-7_plugin_fix_deployment"
                  }
              }
              steps {
                  container('docker') {
                      configFileProvider(
                          [configFile(fileId: '11c739e4-8ac5-4fd3-983a-c20bd29846ef', variable: 'MAVEN_SETTINGS')]) {
                                sh './mvnw -s $MAVEN_SETTINGS clean package deploy'
                      }
                  }
              }
        }


        stage('Build & Publish Develop') {
            when {
                branch "nextflow_22-10-7_plugin_fix_deployment"
            }
            steps {
                container('docker') {
                    withCredentials([usernamePassword(credentialsId:'argoContainers', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')/* ,
                                        usernamePassword(credentialsId: 'argoGithub', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME') */]) {
                       sh 'docker login ghcr.io -u $USERNAME -p $PASSWORD'
                       //sh "docker build --build-arg GH_TOKEN=${GIT_PASSWORD} --build-arg GH_USER=${GIT_USERNAME} --network=host . -t ${dockerRepo}:edge -t ${dockerRepo}:${version}-${commit}"
                    }

                    // DNS error if --network is default
                    sh "docker build --network=host . -t ${dockerRepo}:edge -t ${dockerRepo}:${commit}"

                    sh "docker push ${dockerRepo}:${version}-${commit}"
                    sh "docker push ${dockerRepo}:edge"
                }
            }
        }


        stage('deploy to rdpc-collab-dev') {
            when {
                branch "develop"
            }
            steps {
                build(job: "/provision/update-app-version", parameters: [
                    [$class: 'StringParameterValue', name: 'RDPC_ENV', value: 'dev' ],
                    [$class: 'StringParameterValue', name: 'TARGET_RELEASE', value: 'management'],
                    [$class: 'StringParameterValue', name: 'NEW_APP_VERSION', value: "${version}-${commit}" ]
                ])
            }
        }

        stage('Release & Tag') {
            when {
                branch "master"
            }
            steps {
                container('docker') {
                    withCredentials([usernamePassword(credentialsId: 'argoGithub', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                        sh "git tag ${version}"
                      sh "git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/${gitHubRepo} --tags"
                    }

                    withCredentials([usernamePassword(credentialsId:'argoContainers', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                        sh 'docker login ghcr.io -u $USERNAME -p $PASSWORD'
                    }

                    // DNS error if --network is default
                    sh "docker build --network=host . -t ${dockerRepo}:latest -t ${dockerRepo}:${version}"

                    sh "docker push ${dockerRepo}:${version}"
                    sh "docker push ${dockerRepo}:latest"
                }
            }
        }
        stage('deploy to rdpc-collab-qa') {
            when {
                branch "master"
            }
            steps {
                build(job: "/provision/update-app-version", parameters: [
                    [$class: 'StringParameterValue', name: 'RDPC_ENV', value: 'qa' ],
                    [$class: 'StringParameterValue', name: 'TARGET_RELEASE', value: 'management'],
                    [$class: 'StringParameterValue', name: 'NEW_APP_VERSION', value: "${version}" ]
                ])
            }
        }
    }
}
