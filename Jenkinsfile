pipeline {
  agent any   // we'll run dockerized steps explicitly

  parameters {
    string(name: 'DOCKER_REPO',
           defaultValue: 'yourdockerhubuser/aws-eb-express-sample',
           description: 'Docker Hub repo: user/repo')
    booleanParam(name: 'ENABLE_SNYK', defaultValue: false,
                 description: 'Use Snyk in addition to npm audit')
  }

  options {
    timestamps()
    ansiColor('xterm')
    buildDiscarder(logRotator(daysToKeepStr: '14', numToKeepStr: '25'))
  }

  environment {
    IMAGE_TAG = "${params.DOCKER_REPO}:${env.BUILD_NUMBER}"
  }

  stages {

    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Install & Test (Node 16)') {
      steps {
        script {
          docker.image('node:16-bullseye').inside {
            sh '''
              node -v
              npm -v
              # Use npm ci for reproducible installs; matches assignment intent
              npm ci || npm install
              npm test --if-present
            '''
          }
        }
      }
      post { always { junit testResults: '**/junit*.xml', allowEmptyResults: true } }
    }

    stage('Dependency Scan (npm audit)') {
      steps {
        script {
          docker.image('node:16-bullseye').inside {
            // Fail build on high/critical vulnerabilities
            sh 'npm audit --audit-level=high --production'
          }
        }
      }
    }

    stage('Dependency Scan (Snyk optional)') {
      when { expression { return params.ENABLE_SNYK } }
      environment { SNYK_TOKEN = credentials('SNYK_TOKEN') }
      steps {
        script {
          docker.image('node:16-bullseye').inside {
            sh '''
              npm i -g snyk
              snyk auth "${SNYK_TOKEN}"
              snyk test --severity-threshold=high
            '''
          }
        }
      }
    }

    stage('Build Docker Image') {
      steps {
        sh '''
          docker version
          echo "Building ${IMAGE_TAG}"
          docker build -t "${IMAGE_TAG}" .
        '''
      }
    }

    stage('Push Image') {
      environment { DOCKERHUB = credentials('DOCKERHUB_CREDENTIALS') }
      steps {
        sh '''
          echo "${DOCKERHUB_PSW}" | docker login -u "${DOCKERHUB_USR}" --password-stdin
          docker push "${IMAGE_TAG}"
          docker logout
        '''
      }
    }

    stage('Archive Artifacts') {
      steps {
        sh 'tar czf build-logs.tgz . || true'
        archiveArtifacts artifacts: 'build-logs.tgz, **/*.xml, **/package-lock.json', fingerprint: true
      }
    }
  }

  post {
    success { echo "✅ Success. Pushed: ${IMAGE_TAG}" }
    failure { echo "❌ Failed. Check test/scan logs above." }
  }
}
