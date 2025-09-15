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
            withCredentials([usernamePassword(credentialsId: 'win-share-coreapp', usernameVariable: 'SHARE_USER', passwordVariable: 'SHARE_PASS')]) {
              bat """
              @echo off
              rem Drop any old session to the share
              net use \\\\135.148.26.116\\coreapp /delete /y >nul 2>&1
        
              rem Connect with proper creds
              net use \\\\135.148.26.116\\coreapp %SHARE_PASS% /user:%SHARE_USER% /persistent:no
        
              rem Copy published output (mirror keeps it clean)
              robocopy .\\publish \\\\135.148.26.116\\coreapp /MIR
        
              rem Disconnect
              net use \\\\135.148.26.116\\coreapp /delete /y
              """
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
