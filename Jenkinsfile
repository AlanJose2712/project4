pipeline {
  agent any
  triggers { pollSCM('H/2 * * * *') }
  options { timeout(time: 45, unit: 'MINUTES') }
  environment {
    IMAGE = 'alanjose2929/project4'
    TAG   = "${BUILD_NUMBER}"
  }
  stages {
    stage('Checkout') {
      steps {
        checkout scm
        script {
          def author = sh(script: 'git log -1 --pretty=%an', returnStdout: true).trim()
          env.SKIP = (author == 'jenkins-bot') ? 'true' : 'false'
        }
      }
    }
    stage('SAST - Bandit') {
      when { expression { env.SKIP == 'false' } }
      steps {
        sh '''
          python3 -m venv .venv
          . .venv/bin/activate
          pip install -q bandit pip-audit
          bandit -r project4 -x project4/static,project4/media -f html -o bandit.html || true
          bandit -r project4 -x project4/static,project4/media || true
        '''
      }
    }
    stage('Dependency Scan') {
      when { expression { env.SKIP == 'false' } }
      steps {
        sh '''
          . .venv/bin/activate
          pip-audit -r project4/requirements.txt 2>&1 | tee pip-audit.txt || true
          trivy fs --scanners vuln,secret --severity HIGH,CRITICAL --skip-version-check project4 | tee trivy-fs.txt
        '''
      }
    }
    stage('Build Image') {
      when { expression { env.SKIP == 'false' } }
      steps { sh 'docker build -t $IMAGE:$TAG project4' }
    }
    stage('Image Scan') {
      when { expression { env.SKIP == 'false' } }
      steps { sh 'trivy image --severity CRITICAL --skip-version-check $IMAGE:$TAG | tee trivy-image.txt' }
    }
    stage('Push to Docker Hub') {
      when { expression { env.SKIP == 'false' } }
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
          sh '''
            echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
            docker push $IMAGE:$TAG
          '''
        }
      }
    }
    stage('DAST - ZAP') {
      when { expression { env.SKIP == 'false' } }
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
          timeout(time: 12, unit: 'MINUTES') {
            sh '''
              docker rm -f p4-dast || true
              docker run -d --name p4-dast -p 8001:8000 $IMAGE:$TAG
              sleep 15
              mkdir -p zap-out && chmod 777 zap-out
              docker run --rm --cpus=1 --network host -v $WORKSPACE/zap-out:/zap/wrk zaproxy/zap-stable \
                zap-baseline.py -t http://localhost:8001 -m 1 -T 5 -I -r zap.html
            '''
          }
        }
      }
    }
    stage('Update Manifest (GitOps)') {
      when { expression { env.SKIP == 'false' } }
      steps {
        withCredentials([usernamePassword(credentialsId: 'github-creds', usernameVariable: 'GH_USER', passwordVariable: 'GH_TOKEN')]) {
          sh '''
            sed -i "s|image: .*|image: $IMAGE:$TAG|" k8s/deployment.yaml
            git config user.name jenkins-bot
            git config user.email jenkins-bot@example.com
            git commit -am "Deploy $TAG"
            git push https://$GH_USER:$GH_TOKEN@github.com/sumith-k-sam/project4.git HEAD:main
          '''
        }
      }
    }
  }
  post {
    always {
      sh 'docker rm -f p4-dast || true'
      archiveArtifacts artifacts: 'bandit.html, pip-audit.txt, trivy-*.txt, zap-out/**', allowEmptyArchive: true
    }
  }
}
