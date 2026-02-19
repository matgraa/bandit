pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
  }

  environment {
    DOCKERFILE = "Dockerfile.bandit"

    IMG_DEPS    = "bandit-deps:${BUILD_NUMBER}"
    IMG_BUILDER = "bandit-builder:${BUILD_NUMBER}"
    IMG_TESTER  = "bandit-tester:${BUILD_NUMBER}"

    LOG_DIR = "ci-logs"
  }

  stages {
    stage('Checkout') {
      steps {
        cleanWs()
        checkout scm
      }
    }

    stage('Prepare dirs for logs/artifacts') {
      steps {
        sh '''
          set -euxo pipefail
          mkdir -p "${LOG_DIR}"
        '''
      }
    }

    stage('Build images: deps -> builder -> tester') {
      steps {
        sh '''
          set -euxo pipefail
          docker version

          # 1) Dependencies
          docker build -f "${DOCKERFILE}" --target deps \
            -t "${IMG_DEPS}" .

          # 2) Builder (bazuje na deps w tym samym Dockerfile)
          docker build -f "${DOCKERFILE}" --target builder \
            -t "${IMG_BUILDER}" .

          # 3) Tester (bazuje na builder)
          docker build -f "${DOCKERFILE}" --target tester \
            -t "${IMG_TESTER}" .
        '''
      }
    }

    stage('Run tests in tester container (logs outside)') {
      steps {
        sh '''
          set -euxo pipefail

          # Uruchamiamy testy, ale wymuszamy wyjście pytest do plików na hoście:
          # - junit xml -> do raportu w Jenkinsie
          # - log tekstowy -> jako artefakt
          #
          # Uwaga: kontener w Twoim Dockerfile ma CMD ["pytest","-q"].
          # Nadpisujemy CMD własną komendą, żeby dołożyć parametry raportowania.

          docker run --rm \
            -v "$PWD/${LOG_DIR}:/ci-logs" \
            "${IMG_TESTER}" \
            sh -lc 'pytest -q --junitxml=/ci-logs/junit.xml 2>&1 | tee /ci-logs/pytest.log'
        '''
      }
    }
  }

  post {
    always {
      // Archiwizuj logi jako numerowane artefakty (BUILD_NUMBER jest w ścieżkach powyżej)
      archiveArtifacts artifacts: "${LOG_DIR}/**", fingerprint: true, allowEmptyArchive: true

      // Jeśli junit.xml istnieje — Jenkins pokaże raport testów
      junit testResults: "${LOG_DIR}/junit.xml", allowEmptyResults: true
    }

    cleanup {
      // Sprzątanie lokalnych obrazów (opcjonalne, przydaje się gdy brakuje miejsca)
      sh '''
        set +e
        docker rmi "${IMG_TESTER}" "${IMG_BUILDER}" "${IMG_DEPS}" 2>/dev/null || true
      '''
    }
  }
}
