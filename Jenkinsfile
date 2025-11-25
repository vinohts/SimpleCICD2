pipeline {
    agent any

    parameters {
        string(name: 'S3_BUCKET', defaultValue: 'simplecicd-artifacts-vino', description: 'S3 bucket to upload artifact')
        string(name: 'AWS_REGION', defaultValue: 'ap-south-1', description: 'AWS region for S3')
    }

    environment {
        BUILD_CONFIGURATION = "Release"
        OUTPUT_DIR = "publish"
        ARTIFACT_NAME = "simplecicd2-${BUILD_NUMBER}.zip"
        WEB_PROJECT = "SimpleCICD.WebApp\\SimpleCICD.WebApp.csproj"
        TEST_PROJECT = "SimpleCICD.WebApp.Tests\\SimpleCICD.WebApp.Tests.csproj"
        DEST_DIR = "D:\\temp"
    }

    stages {
        stage('Checkout') { steps { checkout scm } }

        stage('Restore & Build') {
            steps {
                bat 'dotnet --info'
                bat "dotnet restore \"%WEB_PROJECT%\""
                bat "dotnet restore \"%TEST_PROJECT%\""
                bat "dotnet build \"%WEB_PROJECT%\" -c %BUILD_CONFIGURATION% --no-restore"
                bat "dotnet build \"%TEST_PROJECT%\" -c %BUILD_CONFIGURATION% --no-restore"
            }
        }

        stage('Test') {
            steps {
                bat "dotnet test \"%TEST_PROJECT%\" -c %BUILD_CONFIGURATION% --no-build --verbosity normal"
            }
        }

        stage('Publish') {
            steps {
                bat "dotnet publish \"%WEB_PROJECT%\" -c %BUILD_CONFIGURATION% -o %OUTPUT_DIR%"
                bat '''
                    powershell -Command "if (Test-Path %ARTIFACT_NAME%) { Remove-Item %ARTIFACT_NAME% -Force }; Compress-Archive -Path %OUTPUT_DIR%\\* -DestinationPath %ARTIFACT_NAME% -Force"
                '''
            }
        }

        stage('Replace Secret Placeholder') {
            steps {
                // Replace ${SECRET_KEY} (project2 placeholder) with the Jenkins secret MY_API_KEY
                withCredentials([string(credentialsId: 'MY_API_KEY', variable: 'MY_API_KEY')]) {
                    echo "Injecting secret into %OUTPUT_DIR%\\appsettings.json..."
                    // use .Replace to avoid regex and Groovy parsing issues
                    bat '''
                        powershell -Command "$content = Get-Content '%OUTPUT_DIR%\\appsettings.json' -Raw; $content = $content.Replace('${SECRET_KEY}', $env:MY_API_KEY); Set-Content -Path '%OUTPUT_DIR%\\appsettings.json' -Value $content"
                    '''
                    bat '''
                        powershell -Command "if (Test-Path '%ARTIFACT_NAME%') { Remove-Item '%ARTIFACT_NAME%' -Force }; Compress-Archive -Path %OUTPUT_DIR%\\* -DestinationPath %ARTIFACT_NAME% -Force"
                    '''
                }
            }
        }

        stage('Archive Artifact') {
            steps {
                archiveArtifacts artifacts: "${env.ARTIFACT_NAME}", fingerprint: true
            }
        }

        stage('Copy artifact to D:\\\\temp') {
            steps {
                echo "Copying %ARTIFACT_NAME% to %DEST_DIR%"
                bat '''
                    if not exist "%DEST_DIR%" (mkdir "%DEST_DIR%")
                    copy /Y "%WORKSPACE%\\%ARTIFACT_NAME%" "%DEST_DIR%\\%ARTIFACT_NAME%"
                '''
            }
        }

        stage('Upload to S3') {
            steps {
                // requires Pipeline: AWS Steps plugin and an AWS Credentials object named 'aws-credentials-id'
                withAWS(credentials: 'aws-credentials-id', region: "${params.AWS_REGION}") {
                    echo "Uploading %ARTIFACT_NAME% to s3://${params.S3_BUCKET}/"
                    bat '''
                        aws s3 cp "%WORKSPACE%\\%ARTIFACT_NAME%" "s3://%S3_BUCKET%/%ARTIFACT_NAME%" --region %AWS_REGION%
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully. Artifact copied to %DEST_DIR% and uploaded to s3://${params.S3_BUCKET}/%ARTIFACT_NAME%"
        }
        failure {
            echo "Pipeline FAILED — check console output."
        }
    }
}
