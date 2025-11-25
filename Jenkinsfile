pipeline {
  agent any

  parameters {
    string(name: 'S3_BUCKET', defaultValue: 'simplecicd-artifacts-vino', description: 'S3 bucket to upload artifact')
    string(name: 'AWS_REGION', defaultValue: 'ap-south-1', description: 'AWS region')
  }

  environment {
    BUILD_CONFIGURATION = "Release"
    OUTPUT_DIR = "publish"
    ARTIFACT_NAME = "simplecicd2-${BUILD_NUMBER}.zip"
    WEB_PROJECT = "SimpleCICD.WebApp\\SimpleCICD.WebApp.csproj"
    TEST_PROJECT = "SimpleCICD.WebApp.Tests\\SimpleCICD.WebApp.Tests.csproj"
    IIS_DEPLOY_PATH = "C:\\inetpub\\wwwroot\\SimpleCICD2"
  }

  stages {

    stage('Checkout') { steps { checkout scm } }

    stage('Restore') {
      steps {
        bat "dotnet restore \"%WEB_PROJECT%\""
        bat "dotnet restore \"%TEST_PROJECT%\""
      }
    }

    stage('Build') {
      steps {
        bat "dotnet build \"%WEB_PROJECT%\" -c %BUILD_CONFIGURATION% --no-restore"
        bat "dotnet build \"%TEST_PROJECT%\" -c %BUILD_CONFIGURATION% --no-restore"
      }
    }

    stage('Test') {
      steps {
        bat "dotnet test \"%TEST_PROJECT%\" -c %BUILD_CONFIGURATION% --no-build"
      }
    }

    stage('Publish') {
      steps {
        bat "dotnet publish \"%WEB_PROJECT%\" -c %BUILD_CONFIGURATION% -o %OUTPUT_DIR%"
        bat """
          powershell -Command "Compress-Archive -Path ${OUTPUT_DIR}\\\\* -DestinationPath ${ARTIFACT_NAME} -Force"
        """
      }
    }

    stage('Replace Secret Placeholder') {
      steps {
        withCredentials([string(credentialsId: 'MY_API_KEY', variable: 'MY_API_KEY')]) {
          bat """
            powershell -Command "(Get-Content \\"%OUTPUT_DIR%\\appsettings.json\\") -replace '\\$\\{SECRET_KEY\\}', '$env:MY_API_KEY' | Set-Content \\"%OUTPUT_DIR%\\appsettings.json\\""
          """
        }
      }
    }

    stage('Deploy to IIS') {
      steps {
        echo "Deploying to IIS folder: ${IIS_DEPLOY_PATH}"
        bat """
          if not exist "${IIS_DEPLOY_PATH}" mkdir "${IIS_DEPLOY_PATH}"
          xcopy /Y /E \"%WORKSPACE%\\publish\\*\" "${IIS_DEPLOY_PATH}\\"
        """
      }
    }

    stage('Upload to S3') {
      steps {
        withAWS(credentials: 'aws-credentials-id', region: "${params.AWS_REGION}") {
          bat "aws s3 cp \"%WORKSPACE%\\${ARTIFACT_NAME}\" s3://${params.S3_BUCKET}/${ARTIFACT_NAME}"
        }
      }
    }

    stage('Archive Artifact') {
      steps {
        archiveArtifacts artifacts: "${ARTIFACT_NAME}", fingerprint: true
      }
    }

  }

  post {
    success {
      echo "SUCCESS — App deployed to IIS and artifact uploaded to S3!"
    }
    failure {
      echo "FAILED — Check logs"
    }
  }
}
