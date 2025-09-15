pipeline {
  agent any

  // Helpful defaults
  options {
    timestamps()
    ansiColor('xterm')
  }

  // Auto-build when main changes (use githubPush() if you configured a webhook)
  triggers {
    pollSCM('H/5 * * * *')   // every ~5 min; remove if using a webhook
    // githubPush()          // uncomment if you set up the GitHub webhook
  }

  environment {
    // Ensure dotnet is on PATH for the Jenkins service
    DOTNET_ROOT = 'C:\\Program Files\\dotnet'
    PATH = "${DOTNET_ROOT};${env.PATH}"

    // Your web project (avoid solution-level publish warnings)
    PROJECT_PATH = '.\\webapp\\webapp.csproj'

    // SMB share to deploy to
    SHARE_HOST = '135.148.26.116'
    SHARE_NAME = 'coreapp'

    // Jenkins credential ID holding the remote *Windows* admin account
    SHARE_CRED = 'win-share-coreapp-admin'
  }

  stages {
    stage('Checkout') {
      steps {
        cleanWs()
        checkout scm
      }
    }

    stage('Restore') {
      steps {
        bat 'dotnet restore'
      }
    }

    stage('Build') {
      steps {
        bat 'dotnet build -c Release --no-restore'
      }
    }

    stage('Test') {
      steps {
        // Only run tests if a *Tests.csproj exists (prevents false failures)
        powershell '''
          $tests = Get-ChildItem -Recurse -Filter *Tests.csproj | Select-Object -First 1
          if ($tests) {
            Write-Host "Running tests..."
            dotnet test --no-restore -c Release
          } else {
            Write-Host "No test projects found; skipping."
          }
        '''
      }
    }

    stage('Publish') {
      steps {
        bat 'dotnet publish %PROJECT_PATH% -c Release -o .\\publish --no-build'
        archiveArtifacts artifacts: 'publish/**', fingerprint: true, onlyIfSuccessful: true
      }
    }

    stage('Deploy') {
      steps {
        withCredentials([usernamePassword(credentialsId: "${env.SHARE_CRED}", usernameVariable: 'WINUSER', passwordVariable: 'WINPASS')]) {
          bat '''
          @echo off
          setlocal EnableExtensions EnableDelayedExpansion

          set "UNC=\\\\%SHARE_HOST%\\%SHARE_NAME%"

          rem ---- 0) Drop any existing connections to that server (avoid cached creds / 1219) ----
          for /f "tokens=1,2" %%i in ('net use ^| find "\\\\%SHARE_HOST%"') do net use %%j /delete /y >nul 2>&1

          rem ---- 1) Map with the remote Administrator creds ----
          net use %UNC% %WINPASS% /user:%WINUSER% /persistent:no
          if errorlevel 1 (
            echo ERROR: Failed to authenticate as %WINUSER% on %UNC%
            exit /b 1
          )

          rem ---- 2) Copy artifacts (normalize Robocopy exit codes) ----
          robocopy .\\publish %UNC% /MIR /NFL /NDL /NP
          set RC=!ERRORLEVEL!
          if !RC! LSS 8 ( set RC=0 )

          rem ---- 3) Disconnect cleanly (ignore errors) ----
          net use %UNC% /delete /y >nul 2>&1

          exit /b !RC!
          '''
        }
      }
    }
  }

  post {
    success {
      echo '✅ Build, test, publish, and deploy completed successfully.'
    }
    failure {
      echo '❌ Pipeline failed. Check the stage logs above.'
    }
  }
}
