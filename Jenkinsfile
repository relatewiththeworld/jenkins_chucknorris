pipeline {
    agent any
    tools {
        nodejs 'Nodejs 23'
    }
    environment {
        DOCKER_IMAGE = "relatewiththeworld/chucknorrisapp"
        DOCKERFILE = "Dockerfile"
        DOCKER_TAG = "${env.BUILD_NUMBER}"
        EC2_USER = 'ubuntu'
        EC2_HOST = '3.74.162.243'
        CONTAINER_NAME = 'chucknorris-app'
        HOST_PORT = 3000
        CONTAINER_PORT = 3000
        DOCKER_CREDENTIALS_ID = 'dockerhub-creds'
        SSH_CREDENTIALS_ID = 'ec2-ssh-key'
        SONAR_PROJECT_KEY = 'nodejs-app-project'
        SONAR_CREDENTIALS_ID = 'sonar-creds'
        SONAR_HOST_URL = 'http://18.195.244.122:9000'
        NVD_API_KEY = 'NVD_API_KEY'
    }
    
    parameters {
        choice(name: 'SCAN_TYPE', choices: ['Baseline', 'APIS', 'Full'], description: 'Type of scan to perform')
        string(name: 'TARGET', defaultValue: 'http://3.74.162.243:3000/', description: 'Target URL to scan')
        booleanParam(name: 'GENERATE_REPORT', defaultValue: true, description: 'Generate HTML report')
    }
    
    stages {
        stage('Git Checkout') {
            steps {
                git url: 'https://github.com/relatewiththeworld/jenkins_chucknorris.git', branch: 'main'
            }
        }
    
    stage('Install Dependencies') {
        steps {
            sh 'npm install'
        }
    }
    
    stage('Run Test') {
        steps {
            sh 'npm test || true'
        }
    }
    
    stage('SonarQube Analysis') {
     steps {
       withCredentials([string(credentialsId: 'sonar-creds', variable: 'SONAR_CREDENTIALS_ID')]) {
       withSonarQubeEnv('Sonar-Qube') {
       sh '''
          npx sonar-scanner \
            -Dsonar.projectKey=$SONAR_PROJECT_KEY \
            -Dsonar.sources=. \
            -Dsonar.host.url=$SONAR_HOST_URL \
            -Dsonar.login=$SONAR_CREDENTIALS_ID
         '''
      }
    }
  }
}
    
    stage('Dependency-Check Analysis') {
     steps {
       dependencyCheck additionalArguments: """
          --project MyApp
          --scan .
          --format ALL
          --out ./reports
       """, odcInstallation: 'OWASP Dependency-Check'

      dependencyCheckPublisher pattern: 'reports/dependency-check-report.xml'
    }
  }
    
    stage('Docker Build') {
        steps {
            echo 'Building Docker Image'
            script {
                docker.build("${DOCKER_IMAGE}:${DOCKER_TAG}", "-f ${DOCKERFILE} .")
            }
        }
    }
    
    stage('Docker Push') {
      steps {
        echo 'Pushing Docker image to Docker Hub...'
        script {
          docker.withRegistry('https://index.docker.io/v1/', "${DOCKER_CREDENTIALS_ID}") {
          docker.image("${DOCKER_IMAGE}:${DOCKER_TAG}").push()
      }
    }
  }
}
    
    stage('Deploy') {
      steps {
        echo 'Deploying to AWS EC2...'
        script {
          sshagent([SSH_CREDENTIALS_ID]) {
            sh """
              ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} \\
              'docker pull ${DOCKER_IMAGE}:${DOCKER_TAG} && \
              docker stop ${CONTAINER_NAME} || true && \
              docker rm ${CONTAINER_NAME} || true && \
              docker run -d --name ${CONTAINER_NAME} -p ${HOST_PORT}:${CONTAINER_PORT} ${DOCKER_IMAGE}:${DOCKER_TAG}'
               """
          }
        }
      }
    }

stage('Run ZAP Scan') {
    steps {
        script {
            def scanScripts = [
                'Baseline': 'zap-baseline.py',
                'APIS'    : 'zap-api-scan.py',
                'Full'    : 'zap-full-scan.py'
            ]
            def scriptName = scanScripts[params.SCAN_TYPE]
            if (!scriptName) {
                error "Invalid SCAN_TYPE: ${params.SCAN_TYPE}"
            }
            sh """
                docker pull zaproxy/zap-stable
                chmod 777 \$(pwd)
                docker run --rm -v \$(pwd):/zap/wrk/:rw zaproxy/zap-stable ${scriptName} -t ${params.TARGET} -r report.html -I
            """
        }
    }
    }

stage('Archive Report') {
            when {
                expression { return params.GENERATE_REPORT }
            }
            steps {
                archiveArtifacts artifacts: 'report.html', allowEmptyArchive: true
            }
        }

}
}