pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
  }

  environment {
    DOTNET_CLI_TELEMETRY_OPTOUT = '1'
    DOTNET_SKIP_FIRST_TIME_EXPERIENCE = '1'
    CONFIGURATION = 'Release'
    TEST_RESULTS_DIR = 'TestResults'
    // If you have a configured Jenkins tool for .NET 6, uncomment next line and set its name
    // DOTNET_HOME = tool name: 'dotnet6'
  }

  stages {
    stage('Branch Gate') {
      when {
        not {
          anyOf {
            branch 'main'
            branch pattern: "feature/.*", comparator: "REGEXP"
          }
        }
      }
      steps {
        echo "Branch '${env.BRANCH_NAME}' not in [main, feature/*] — skipping."
        script { currentBuild.result = 'NOT_BUILT' }
      }
    }

    stage('Checkout') {
      when {
        anyOf {
          branch 'main'
          branch pattern: "feature/.*", comparator: "REGEXP"
        }
      }
      steps {
        checkout scm
      }
    }

    stage('.NET SDK Info') {
      when {
        anyOf {
          branch 'main'
          branch pattern: "feature/.*", comparator: "REGEXP"
        }
      }
      steps {
        script {
          if (isUnix()) {
            sh 'dotnet --info'
          } else {
            bat 'dotnet --info'
          }
        }
      }
    }

    stage('Restore') {
      when {
        anyOf {
          branch 'main'
          branch pattern: "feature/.*", comparator: "REGEXP"
        }
      }
      steps {
        script {
          if (isUnix()) {
            sh 'dotnet restore'
          } else {
            bat 'dotnet restore'
          }
        }
      }
    }

    stage('Build') {
      when {
        anyOf {
          branch 'main'
          branch pattern: "feature/.*", comparator: "REGEXP"
        }
      }
      steps {
        script {
          if (isUnix()) {
            sh 'dotnet build --configuration $CONFIGURATION --no-restore'
          } else {
            bat 'dotnet build --configuration %CONFIGURATION% --no-restore'
          }
        }
      }
    }

    stage('Test') {
      when {
        anyOf {
          branch 'main'
          branch pattern: "feature/.*", comparator: "REGEXP"
        }
      }
      steps {
        script {
          if (isUnix()) {
            sh '''
              rm -rf "$TEST_RESULTS_DIR"
              mkdir -p "$TEST_RESULTS_DIR"
              dotnet test --configuration "$CONFIGURATION" --no-build \
                --logger trx \
                --results-directory "$TEST_RESULTS_DIR" \
                /p:CollectCoverage=true \
                /p:CoverletOutputFormat=cobertura \
                /p:CoverletOutput="$TEST_RESULTS_DIR/coverage/"
            '''
          } else {
            bat '''
              if exist "%TEST_RESULTS_DIR%" rmdir /s /q "%TEST_RESULTS_DIR%"
              mkdir "%TEST_RESULTS_DIR%"
              dotnet test --configuration "%CONFIGURATION%" --no-build ^
                --logger trx ^
                --results-directory "%TEST_RESULTS_DIR%" ^
                /p:CollectCoverage=true ^
                /p:CoverletOutputFormat=cobertura ^
                /p:CoverletOutput="%TEST_RESULTS_DIR%\\coverage\\"
            '''
          }
        }
      }
      post {
        always {
          // Publish TRX test results (requires xUnit plugin; uses MSTest reader for TRX)
          catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
            xunit(
              thresholds: [ skipped(failureThreshold: '0') ],
              tools: [ MSTest(pattern: "${TEST_RESULTS_DIR}/**/*.trx") ]
            )
          }
          // Publish coverage if Cobertura report is present (requires Cobertura plugin)
          script {
            def cov = findFiles(glob: "${TEST_RESULTS_DIR}/**/coverage.cobertura.xml")
            if (cov && cov.length > 0) {
              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                cobertura coberturaReportFile: "${TEST_RESULTS_DIR}/**/coverage.cobertura.xml",
                         autoUpdateHealth: false, autoUpdateStability: false,
                         failNoReports: false, maxNumberOfBuilds: 0,
                         onlyStable: false, sourceEncoding: 'UTF_8'
              }
            } else {
              echo "No Cobertura coverage report found."
            }
          }
          archiveArtifacts artifacts: "${TEST_RESULTS_DIR}/**/*", allowEmptyArchive: true
        }
      }
    }
  }

  post {
    success { echo "✅ Build & tests passed on ${env.BRANCH_NAME}" }
    failure { echo "❌ Build or tests failed on ${env.BRANCH_NAME}" }
  }
}
