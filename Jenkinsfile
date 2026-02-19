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

    stage('Build images (deps -> build -> tester)') {
      steps {
        script {
          // tag: BUILD_NUMBER + short SHA (jak chciałeś)
          env.TAG        = "${BUILD_NUMBER}-${GIT_SHA}"

          env.IMG_DEPS   = "bandit-deps:${TAG}"
          env.IMG_BUILD  = "bandit-build:${TAG}"
          env.IMG_TESTER = "bandit-tester:${TAG}"

          // PEP 440 compliant (ważne!)
          env.PBR_VER    = "0.0.0+${BUILD_NUMBER}.${GIT_SHA}"
        }

        sh '''
          set -euxo pipefail
          docker version

          docker build -f "${DOCKERFILE}" --target deps \
            -t "${IMG_DEPS}" .

          docker build -f "${DOCKERFILE}" --target build \
            --build-arg PBR_VERSION="${PBR_VER}" \
            -t "${IMG_BUILD}" .

          docker build -f "${DOCKERFILE}" --target tester \
            --build-arg PBR_VERSION="${PBR_VER}" \
            -t "${IMG_TESTER}" .
        '''
      }
    }

    stage('Test (tester runs, logs outside)') {
      steps {
        sh '''
          set -euxo pipefail

          docker run --rm \
            -e PBR_VERSION="${PBR_VER}" \
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

      // JUnit zadziała dopiero gdy wygenerujesz junit.xml (np. z tox/pytest/stestr)
      junit testResults: "${LOG_DIR}/junit.xml", allowEmptyResults: true
    }
  }
}
