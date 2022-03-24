// Generate a random id for pod label to avoid waiting for executor. Just start create pod right away.
// See this workaround from issue: https://issues.jenkins.io/browse/JENKINS-39801
import static java.util.UUID.randomUUID
def uuid = randomUUID() as String
def myid = uuid.take(8)

pipeline {
  environment {
    APP_VER = "v1.0.${BUILD_ID}"
    // HARBOR_URL = ""
    DEPLOY_GITREPO_USER = "mridula2"    
    DEPLOY_GITREPO_URL = "github.com/${DEPLOY_GITREPO_USER}/react-dashboard-helmchart.git"
    DEPLOY_GITREPO_BRANCH = "main"
    DEPLOY_GITREPO_TOKEN = credentials('mri-github')
  }    
  agent {
    kubernetes {
      label "react-dashboard-${myid}"
      instanceCap 1
      defaultContainer 'jnlp'
      yaml """
apiVersion: v1
kind: Pod
metadata:
  namespace: jenkins-workers
spec:
  # Use service account that can deploy to all namespaces
  serviceAccountName: default
  containers:
  - name: nodejs
    image: zarakmughal/nodejs-starter:1.0
    command:
    - cat
    tty: true
    volumeMounts:
      - mountPath: "/root/.m2"
        name: m2
  - name: kaniko
    image: gcr.io/kaniko-project/executor:v1.6.0-debug
    imagePullPolicy: Always
    command:
    - sleep
    args:
    - 99d    
    volumeMounts:
    - mountPath: "/root/.m2"
      name: m2      
    - name: docker-config
      mountPath: /kaniko/.docker
    - name: ca-cert
      mountPath: /kaniko/ssl/certs/
  volumes:
    - name: ca-cert
      secret:
        secretName: ca-bundle
        items:
        - key: additional-ca-cert-bundle.crt
          path: additional-ca-cert-bundle.crt
    - name: docker-config
      configMap:
        name: docker-config
    - name: m2
      persistentVolumeClaim:
        claimName: m2
"""
}
   }
  stages {
    stage('Build') {
      steps {
        container('nodejs') {
          echo sh(script: 'env|sort', returnStdout: true)
          sh """
            npm install
            """
        }
      }
    }
    stage('Test') {
      parallel {
        stage(' Unit/Integration Tests') {
          steps {
            echo "Unit/Integration test"
//             container('nodejs') {
//               sh """
//                 npm test
//               """
//             }
//             jacoco ( 
//               execPattern: 'target/*.exec',
//               classPattern: 'target/classes',
//               sourcePattern: 'src/main/java',
//               exclusionPattern: 'src/test*'
//             )
          }
//           post {
//             always {
//               archiveArtifacts artifacts: 'target/**/*.jar', fingerprint: true
//               junit 'target/surefire-reports/**/*.xml'
//             }
//           }           
        }
        stage('Static Code Analysis') {
          steps {
            echo "Sonarcube test"
//             container('nodejs') {
//               withSonarQubeEnv('My SonarQube') { 
//                 sh """
//                 mvn sonar:sonar \
//                   -Dsonar.projectKey=react-dashboard \
//                   -Dsonar.host.url=${env.SONAR_HOST_URL} \
//                   -Dsonar.login=${env.SONAR_AUTH_TOKEN}
//                 """
//               }
//             }
          }          
        }            
      }
    }
    stage('Containerize') {
      steps {
        container('kaniko') {
          sh "sed -i 's,harbor.example.com,${env.HARBOR_URL},g' Dockerfile"
          sh "/kaniko/executor --dockerfile Dockerfile --context `pwd` --skip-tls-verify --destination=${env.HARBOR_URL}/library/samples/react-dashboard:v1.0.${env.BUILD_ID}"
        }
      }
    }
    stage('Image Vulnerability Scan') {
      steps {
        echo "Image Vulnerability Scan"
//         writeFile file: 'anchore_images', text: "${env.HARBOR_URL}/library/samples/react-dashboard:v1.0.${env.BUILD_ID}"
//         anchore name: 'anchore_images'
      }
    }
    stage('Approval') {
      input {
        message "Proceed to deploy?"
        ok "YES"
      }
      steps {
        echo "Update helm chart to trigger GitOps-based deployment..."
      }
    }    
    stage('GitOps-based Deploy') {
      steps {
        container('nodejs') {
          sh """
            git config --global user.name $env.GIT_AUTHOR_NAME
            git config --global user.email $env.GIT_AUTHOR_EMAIL
            git clone https://$env.DEPLOY_GITREPO_USER:$env.DEPLOY_GITREPO_TOKEN@$env.DEPLOY_GITREPO_URL --branch=$env.DEPLOY_GITREPO_BRANCH deploy
            # After cloning
            cd deploy
            # update values.yaml
            sed -i -r 's,repository: (.+),repository: ${env.HARBOR_URL}/library/samples/react-dashboard,' values.yaml
            sed -i 's/tag: v1.0.*/tag: v1.0.${env.BUILD_ID}/' values.yaml
            cat values.yaml
            git commit -am 'bump up version number'
            git remote set-url origin https://$env.DEPLOY_GITREPO_USER:$env.DEPLOY_GITREPO_TOKEN@$env.DEPLOY_GITREPO_URL
            git push origin main
          """
        }
      }
    }   
  }
}
