pipeline {
    agent any

    environment {
        DOTNET_CLI_HOME = "C:\\Program Files\\dotnet"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                script {
                    // Restoring dependencies
                    //bat "cd ${DOTNET_CLI_HOME} && dotnet restore"
                    bat "dotnet restore"

                    // Building the application
                    bat "dotnet build --configuration Release"
                }
            }
        }

        stage('Test') {
            steps {
                script {
                    // Running tests
                    bat "dotnet test --no-restore --configuration Release"
                }
            }
        }

        stage('Publish') {
            steps {
                script {
                    // Publishing the application
                    bat "dotnet publish --no-restore --configuration Release --output .\\publish"
                }
            }
        }
        stage('Deploy') {
          steps {
            withCredentials([usernamePassword(credentialsId: 'win-share-coreapp-admin', usernameVariable: 'WINUSER', passwordVariable: 'WINPASS')]) {
              bat '''
              @echo off
              setlocal EnableExtensions
        
              rem ---- 0) Drop any existing connections to that server (avoid cached creds / 1219) ----
              for /f "tokens=1,2" %%i in ('net use ^| find "\\\\135.148.26.116"') do net use %%j /delete /y >nul 2>&1
        
              rem ---- 1) Map with the remote Administrator creds ----
              net use \\\\135.148.26.116\\coreapp %WINPASS% /user:%WINUSER% /persistent:no
              if errorlevel 1 (
                echo ERROR: Failed to authenticate as %WINUSER% on \\\\135.148.26.116\\coreapp
                exit /b 1
              )
        
              rem ---- 2) Copy artifacts (treat Robocopy codes <8 as success) ----
              robocopy .\\publish \\\\135.148.26.116\\coreapp /MIR /NFL /NDL /NP
              set RC=%ERRORLEVEL%
              if %RC% LSS 8 ( set RC=0 )
        
              rem ---- 3) Disconnect cleanly ----
              net use \\\\135.148.26.116\\coreapp /delete /y >nul 2>&1
        
              exit /b %RC%
              '''
            }
          }
        }

    }

    post {
        success {
            echo 'Build, test, and publish successful!'
        }
    }
}
