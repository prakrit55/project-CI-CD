def label = "jenkinsagent"
def mvn_version = 'M2'
podTemplate(label: label, yaml: """
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: build
  annotations:
    sidecar.istio.io/inject: "false"
spec:
  containers:
  - name: build
    image: prakrit55/jenkinsagent
    command:
    - cat
    tty: true
    volumeMounts:
    - name: dockersock
      mountPath: /var/run/docker.sock
  volumes:
  - name: dockersock
    hostPath:
      path: /var/run/docker.sock
"""
) {
    node (label) {
        stage ('Checkout SCM'){
          git credentialsId: 'git', url: 'https://github.com/prakrit55/project-CI-CD', branch: 'master'
          container('build') {
                stage('Build a Maven project') {
                  //withEnv( ["PATH+MAVEN=${tool mvn_version}/bin"] ) {
                   //sh "mvn clean package"
                  //  }
                  sh './mvnw clean package' 
                   //sh 'mvn clean package'
                }
            }
        }
        stage ('Sonar Scan'){
          container('build') {
                stage('Sonar Scan') {
                  withSonarQubeEnv('sonar') {
                  sh './mvnw verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=etccompany_etccompany'
                }
                }
            }
        }


        stage ('Artifactory configuration'){
          container('build') {
                stage('Artifactory configuration') {
                    rtServer (
                    id: "jfrog",
                    url: "https://etccompany.jfrog.io/artifactory",
                    credentialsId: "jfrog"
                )

                rtMavenDeployer (
                    id: "MAVEN_DEPLOYER",
                    serverId: "jfrog",
                    releaseRepo: "etccompany-libs-release-local",
                    snapshotRepo: "etccompany-libs-release-local"
                )

                rtMavenResolver (
                    id: "MAVEN_RESOLVER",
                    serverId: "jfrog",
                    releaseRepo: "etccompany-libs-release",
                    snapshotRepo: "etccompany-libs-release"
                )            
                }
            }
        }
        stage ('Deploy Artifacts'){
          container('build') {
                stage('Deploy Artifacts') {
                    rtMavenRun (
                    tool: "java", // Tool name from Jenkins configuration
                    useWrapper: true,
                    pom: 'pom.xml',
                    goals: 'clean install',
                    deployerId: "MAVEN_DEPLOYER",
                    resolverId: "MAVEN_RESOLVER"
                  )
                }
            }
        }
        stage ('Publish build info') {
            container('build') {
                stage('Publish build info') {
                rtPublishBuildInfo (
                    serverId: "jfrog"
                  )
               }
           }
       }
       stage ('Docker Build'){
          container('build') {
                stage('Build Image') {
                    docker.withRegistry( 'https://registry.hub.docker.com', 'docker' ) {
                    def customImage = docker.build("prakrit55/etccompany-micro-services-admin:latest")
                    customImage.push()             
                    }
                }
            }
        }

        stage ('Helm Chart') {
          container('build') {
            dir('charts') {
              withCredentials([usernamePassword(credentialsId: 'jfrog', usernameVariable: 'username', passwordVariable: 'password')]) {
              sh '/usr/local/bin/helm package micro-services-admin'
              sh '/usr/local/bin/helm push-artifactory micro-services-admin-1.0.tgz https://etccompany.jfrog.io/artifactory/etccompany-helm-local --username $username --password $password'
              }
            }
        }
        }
    }
}
