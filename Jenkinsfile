pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
    skipDefaultCheckout(true)
  }

  environment {
    DOCKERFILE      = "Dockerfile.bandit"
    LOG_DIR         = "ci-logs"
    DOCKER_BUILDKIT = "1"
  }

  stages {
    stage('Checkout') {
      steps {
        cleanWs()
        checkout scm
        script {
          env.GIT_SHA = sh(returnStdout: true, script: "git rev-parse --short HEAD").trim()
        }
      }
    }

    stage('Prepare log dir') {
      steps {
        sh 'set -euxo pipefail; mkdir -p "${LOG_DIR}"'
      }
    }

    stage('Build images (deps -> builder -> tester)') {
      steps {
        script {
          env.IMG_DEPS    = "bandit-deps:${BUILD_NUMBER}-${GIT_SHA}"
          env.IMG_BUILDER = "bandit-builder:${BUILD_NUMBER}-${GIT_SHA}"
          env.IMG_TESTER  = "bandit-tester:${BUILD_NUMBER}-${GIT_SHA}"
        }

        sh '''
          set -euxo pipefail
          docker version

          docker build -f "${DOCKERFILE}" --target deps    -t "${IMG_DEPS}" .
          docker build -f "${DOCKERFILE}" --target builder -t "${IMG_BUILDER}" .
          docker build -f "${DOCKERFILE}" --target tester  -t "${IMG_TESTER}" .
        '''
      }
    }

    stage('Test (inside tester, logs outside)') {
      steps {
        sh '''
          set -euxo pipefail

          # Uwaga: tox zwykle sam zarządza środowiskiem.
          # Tu zapisujemy log do pliku na hoście.
          docker run --rm \
            -v "$PWD/${LOG_DIR}:/ci-logs" \
            "${IMG_TESTER}" \
            sh -lc 'tox -q 2>&1 | tee /ci-logs/tox.log'
        '''
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: "${LOG_DIR}/**", fingerprint: true, allowEmptyArchive: true
      // jeśli nie generujesz junit xml, to junit będzie puste i OK (allowEmptyResults)
      junit testResults: "${LOG_DIR}/junit.xml", allowEmptyResults: true
    }

    cleanup {
      sh '''
        set +e
        # Zostawienie obrazów poprawia cache. Jeśli musisz czyścić, odkomentuj:
        # docker rmi "${IMG_TESTER}" "${IMG_BUILDER}" "${IMG_DEPS}" 2>/dev/null || true
        true
      '''
    }
  }
}
