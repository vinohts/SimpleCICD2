pipeline {
    agent any

    parameters {
        string(name: 'S3_BUCKET', defaultValue: 'simplecicd-artifacts-vino', description: 'S3 bucket to upload artifact')
        string(name: 'AWS_REGION', defaultValue: 'ap-south-1', description: 'AWS region for SecretsManager & S3')
        string(name: 'SECRET_NAME', defaultValue: 'simplecicd/app-secrets', description: 'AWS Secrets Manager secret name (or ARN)')
        string(name: 'SECRET_JSON_FIELD', defaultValue: 'ApiKey', description: 'If secret is JSON, the field to extract; leave blank if secret is plain string')
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
                    powershell -Command "if (Test-Path '%ARTIFACT_NAME%') { Remove-Item '%ARTIFACT_NAME%' -Force }; Compress-Archive -Path %OUTPUT_DIR%\\* -DestinationPath %ARTIFACT_NAME% -Force"
                '''
            }
        }

        stage('Fetch secret & Replace Placeholder') {
            steps {
                // requires Pipeline: AWS Steps plugin and an AWS Credentials object named 'aws-credentials-id'
                withAWS(credentials: 'aws-credentials-id', region: "${params.AWS_REGION}") {
                    echo "Fetching secret '${params.SECRET_NAME}' from Secrets Manager..."
                    bat '''
                        powershell -NoProfile -Command ^
                          "$secretJson = (aws secretsmanager get-secret-value --secret-id '%SECRET_NAME%' --region %AWS_REGION% --query SecretString --output text) ; " ^
                          "if (-not $secretJson) { Write-Error 'Secret fetch failed or returned empty'; exit 1 } ; " ^
                          "if ('%SECRET_JSON_FIELD%' -ne '') { $obj = $secretJson | ConvertFrom-Json ; $secretValue = $obj.%SECRET_JSON_FIELD% } else { $secretValue = $secretJson } ; " ^
                          "if (-not $secretValue) { Write-Error 'Secret field not found or empty'; exit 1 } ; " ^
                          "$pub = '%OUTPUT_DIR%\\appsettings.json' ; " ^
                          "if (-not (Test-Path $pub)) { Write-Error ('Publish appsettings.json not found at ' + $pub); exit 1 } ; " ^
                          "$content = Get-Content $pub -Raw ; " ^
                          "$new = $content.Replace('${SECRET_KEY}', $secretValue) ; " ^
                          "Set-Content -Path $pub -Value $new ; " ^
                          "Exit 0"
                    '''
                    // recreate ZIP so it contains the replaced appsettings.json
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
                withAWS(credentials: 'aws-credentials-id', region: "${params.AWS_REGION}") {
                    echo "Uploading %ARTIFACT_NAME% to s3://${params.S3_BUCKET}/"
                    bat '''
                        if not exist "%WORKSPACE%\\%ARTIFACT_NAME%" (
                          echo Artifact not found: %WORKSPACE%\\%ARTIFACT_NAME%
                          exit /b 1
                        )
                        aws s3 cp "%WORKSPACE%\\%ARTIFACT_NAME%" "s3://%S3_BUCKET%/%ARTIFACT_NAME%" --region %AWS_REGION%
                    '''
                }
            }
        }
    }

    post {
        success { echo "Pipeline finished: artifact uploaded to s3://${params.S3_BUCKET}/${ARTIFACT_NAME}" }
        failure { echo "Pipeline failed — check console logs." }
    }
}
