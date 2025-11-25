pipeline {
    agent any

    parameters {
        string(name: 'S3_BUCKET', defaultValue: 'simplecicd-artifacts-vino', description: 'S3 bucket to upload artifact')
        string(name: 'AWS_REGION', defaultValue: 'ap-south-1', description: 'AWS region for S3')
    }

    environment {
        BUILD_CONFIGURATION = "Release"
        OUTPUT_DIR = "publish"
        ARTIFACT_NAME = "simplecicd.zip"
        API_PROJECT = "SimpleCICD.Api\\SimpleCICD.Api.csproj"
        TEST_PROJECT = "SimpleCICD.Tests\\SimpleCICD.Tests.csproj"
        DEST_DIR = "D:\\temp"
    }

    stages {
        stage('Checkout') { steps { checkout scm } }

        stage('Restore & Build') {
            steps {
                bat 'dotnet --info'
                bat "dotnet restore \"%API_PROJECT%\""
                bat "dotnet restore \"%TEST_PROJECT%\""
                bat "dotnet build \"%API_PROJECT%\" -c %BUILD_CONFIGURATION% --no-restore"
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
                bat "dotnet publish \"%API_PROJECT%\" -c %BUILD_CONFIGURATION% -o %OUTPUT_DIR%"
                bat '''
                    powershell -Command "if (Test-Path 'simplecicd.zip') { Remove-Item 'simplecicd.zip' -Force }; Compress-Archive -Path %OUTPUT_DIR%\\* -DestinationPath simplecicd.zip -Force"
                '''
            }
        }

        stage('Replace Secret Placeholder') {
            steps {
                withCredentials([string(credentialsId: 'MY_API_KEY', variable: 'MY_API_KEY')]) {
                    echo "Injecting Jenkins secret into %OUTPUT_DIR%\\appsettings.json..."
                    bat '''
                        powershell -Command "$content = Get-Content '%OUTPUT_DIR%\\appsettings.json' -Raw; $content = $content.Replace('${API_KEY_PLACEHOLDER}', $env:MY_API_KEY); Set-Content -Path '%OUTPUT_DIR%\\appsettings.json' -Value $content"
                    '''
                    bat '''
                        powershell -Command "if (Test-Path 'simplecicd.zip') { Remove-Item 'simplecicd.zip' -Force }; Compress-Archive -Path %OUTPUT_DIR%\\* -DestinationPath simplecicd.zip -Force"
                    '''
                }
            }
        }

        stage('Archive Artifact') {
            steps {
                archiveArtifacts artifacts: "${ARTIFACT_NAME}", fingerprint: true
            }
        }

        stage('Copy artifact to D:\\temp') {
            steps {
                echo "Copying ${ARTIFACT_NAME} to ${env.DEST_DIR}"
                bat '''
                    if not exist "%DEST_DIR%" (mkdir "%DEST_DIR%")
                    copy /Y "%WORKSPACE%\\simplecicd.zip" "%DEST_DIR%\\simplecicd.zip"
                '''
            }
        }

        stage('Upload to S3') {
            steps {
                withAWS(credentials: 'aws-credentials-id', region: "${params.AWS_REGION}") {
                    echo "Uploading ${ARTIFACT_NAME} to s3://${params.S3_BUCKET}/"
                    bat '''
                        aws s3 cp "%WORKSPACE%\\simplecicd.zip" "s3://%S3_BUCKET%/simplecicd.zip" --region %AWS_REGION%
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully. Artifact copied to ${env.DEST_DIR} and uploaded to s3://${params.S3_BUCKET}/${ARTIFACT_NAME}"
        }
        failure {
            echo "Pipeline FAILED — check console output."
        }
    }
}
