pipeline {
  agent any

  // Safe, built-in options only
  options {
    timestamps()
  }

  // Build + deploy settings
  environment {
    // Make sure Jenkins service can find dotnet
    DOTNET_ROOT = 'C:\\Program Files\\dotnet'
    PATH = "${DOTNET_ROOT};${env.PATH}"

    // Your web project (publish this csproj to avoid solution-level warnings)
    PROJECT_PATH = '.\\webapp\\webapp.csproj'

    // SMB deploy target
    SHARE_HOST = '135.148.26.116'
    SHARE_NAME = 'coreapp'

    // Jenkins credential ID for the *remote* Windows admin (e.g., SERVERNAME\Administrator)
    SHARE_CRED = 'win-share-coreapp-admin'
  }

  // Enable this if you want auto-builds without a webhook
  // triggers { pollSCM('H/5 * * * *') }

  stages {
    stage('Checkout') {
      steps {
        // built-in cleanup (no plugin needed)
        deleteDir()
        checkout scm
      }
    }

    stage('Restore') {
      steps {
        // restore from repo root (works with solution or single project)
        bat 'dotnet restore'
      }
    }

    stage('Build') {
      steps {
        // build (fast) — no-restore since we already restored
        bat 'dotnet build -c Release --no-restore'
      }
    }

    stage('Test') {
      steps {
        // Run tests only if any *Tests.csproj exists to avoid false failures
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
        // publish the *project* (no solution-level /o warning)
        bat 'dotnet publish "%PROJECT_PATH%" -c Release -o .\\publish --no-build'
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

          rem ---- 0) Drop any existing connections to that server (ignore errors) ----
          for /f "tokens=1,2" %%i in ('net use ^| find "\\\\%SHARE_HOST%"') do net use %%j /delete /y >nul 2>&1

          rem ---- 1) Map with the remote Windows creds (e.g., SERVERNAME\\Administrator) ----
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
      echo '✅ Build, publish, and deploy completed successfully.'
    }
    failure {
      echo '❌ Pipeline failed. Check the stage logs above.'
    }
  }
}
