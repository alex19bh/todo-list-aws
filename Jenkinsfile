pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'
        STACK_NAME = 'todo-list-aws-staging'
        S3_BUCKET = 'aws-sam-cli-managed-default-samclisourcebucket-qmu3tffcy8al'
        SAM_TEMPLATE = 'template.yaml'
    }

    stages {
        stage('Get Code') {
            steps {
                cleanWs()
                echo 'Descargando código desde la rama develop...'
                git branch: 'develop', url: 'https://github.com/alex19bh/todo-list-aws.git'
            }
        }

        stage('Static Test') {
            steps {
                echo 'Ejecutando Flake8 y Bandit...'
                sh '''
                    flake8 src > flake8-report.txt || true
                    bandit -r src -f txt -o bandit-report.txt || true
                '''
                recordIssues tools: [
                    flake8(pattern: 'flake8-report.txt'),
                    pyLint(name: 'Bandit', pattern: 'bandit-report.txt')
                ]
            }
        }

        stage('Deploy') {
            steps {
                echo 'Desplegando con SAM en entorno Staging...'
                sh '''
                    sam build
                    sam deploy --stack-name $STACK_NAME --s3-bucket $S3_BUCKET --region $AWS_REGION --capabilities CAPABILITY_IAM
                '''
            }
        }

        stage('Rest Test') {
            steps {
                echo 'Ejecutando pruebas de integración con pytest...'
                sh '''
                    export API_URL=$(aws cloudformation describe-stacks --stack-name $STACK_NAME --query "Stacks[0].Outputs[?OutputKey=='ApiUrl'].OutputValue" --output text)
                    echo "Probando la API en: $API_URL"
                    if [ -z "$API_URL" ]; then
                        echo "No se pudo obtener la URL de la API del stack"; exit 1
                    fi
                    pytest test/integration/todoApiTest.py --api-url=$API_URL || exit 1
                '''
            }
        }

        stage('Promote') {
            when {
                expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
            }
            steps {
                echo 'Mergeando develop → master...'
                sh '''
                    git config user.name "alex19bh"
                    git config user.email "alejandro.ortiz.salduba@gmail.com"
                    git fetch origin
                    git checkout master
                    git merge origin/develop
                    git push origin master
                '''
            }
        }
    }
}
