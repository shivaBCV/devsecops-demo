pipeline {
  agent any

  options {
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }

  triggers {
    pollSCM('H/2 * * * *')
  }

  environment {
    IMAGE = "demo-app:${BUILD_NUMBER}"
  }

  stages {

    stage('Commit info') {
      steps {
        sh 'git log -1 --pretty="Commit %h by %an: %s"'
      }
    }

    stage('Trivy: code & dependency scan') {
      steps {
        sh '''
          trivy fs --scanners vuln,misconfig,secret --format json -o trivy-fs.json .
          trivy convert --format table trivy-fs.json
        '''
      }
    }

    stage('Build image') {
      steps {
        sh 'docker build -t "$IMAGE" .'
      }
    }

    stage('Trivy: image scan') {
      steps {
        sh '''
          trivy image --format json -o trivy-image.json "$IMAGE"
          trivy convert --format table --severity HIGH,CRITICAL trivy-image.json
        '''
      }
    }

    stage('Send findings to Wazuh') {
      steps {
        sh '''
          LOG=/var/log/trivy/trivy-results.json
          before=$(wc -l < "$LOG")
          for f in trivy-fs.json trivy-image.json; do
            jq -c --arg job "$JOB_NAME" --arg build "$BUILD_NUMBER" --arg commit "$GIT_COMMIT" '
              .Results[]? | .Target as $t | .Vulnerabilities[]?
              | select(.Severity == "HIGH" or .Severity == "CRITICAL")
              | {tool:"trivy", job:$job, build:$build, commit:$commit, target:$t,
                 vuln_id:.VulnerabilityID, pkg:.PkgName,
                 installed:.InstalledVersion, fixed:(.FixedVersion // "none"),
                 severity:.Severity, title:(.Title // "")}
            ' "$f" >> "$LOG"
          done
          after=$(wc -l < "$LOG")
          echo "Findings sent to Wazuh: $((after - before))"
        '''
      }
    }

    stage('Security gate') {
      steps {
        sh 'trivy image --exit-code 1 --severity CRITICAL --ignore-unfixed "$IMAGE"'
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'trivy-*.json', allowEmptyArchive: true
      sh 'docker rmi "$IMAGE" || true'
    }
    success { echo 'No fixable CRITICAL vulnerabilities. Build passed the security gate.' }
    failure { echo 'Build failed. Check which stage failed in the console output.' }
  }
}
