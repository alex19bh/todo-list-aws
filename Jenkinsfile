pipeline {
    agent none

    environment {
        AWS_REGION = 'us-east-1'
        STACK_NAME = 'todo-list-aws-production'
        S3_BUCKET = 'aws-sam-cli-managed-default-samclisourcebucket-qmu3tffcy8al'
        REPO = 'todo-list-aws.git'
        USUARIO_GITHUB = 'alex19bh'
        CONFIG_REPO = 'todo-list-aws-config'
    }

    stages {
        stage('Get Code') {
            agent { label 'principal' }
            steps {
                cleanWs()
                echo 'Descargando código desde la rama master...'
                git branch: 'master', url: "https://github.com/${USUARIO_GITHUB}/${REPO}"

                script {
                    echo "Descargando samconfig.toml desde rama production de repo ${CONFIG_REPO}"
                    sh '''
                        git init config
                        cd config
                        git remote add origin https://github.com/${USUARIO_GITHUB}/${CONFIG_REPO}.git
                        git config core.sparseCheckout true
                        echo "samconfig.toml" > .git/info/sparse-checkout
                        git pull origin production
                        cd ..
                        mv config/samconfig.toml ./samconfig.toml
                        rm -rf config
                    '''
                }
                stash name: 'fuente', includes: '**/*'
            }
        }

        stage('Deploy') {
            agent { label 'principal' }
            steps {
                cleanWs()
                unstash 'fuente'
                sh '''
                    sam build
                    sam deploy --config-file samconfig.toml --config-env production --no-fail-on-empty-changeset
                '''
                script {
                    env.BASE_URL = sh(
                        script: """
                            aws cloudformation describe-stacks \
                              --stack-name ${STACK_NAME} \
                              --query "Stacks[0].Outputs[?OutputKey=='BaseUrlApi'].OutputValue" \
                              --region ${AWS_REGION} --output text
                        """,
                        returnStdout: true
                    ).trim()
                    echo "Base URL obtenida: ${env.BASE_URL}"
                }
                stash name: 'fuente', includes: '**/*'
            }
        }

        stage('Rest Test') {
            agent { label 'tercernodoubuntu' }
            steps {
                cleanWs()
                unstash 'fuente'
                sh '''
                    echo "Usando BASE_URL=$BASE_URL"
                    export BASE_URL
                    sleep 10
                    python3 -m pytest -k "get or list" test/integration/todoApiTest.py || exit 1
                '''
            }
        }
    }
}
