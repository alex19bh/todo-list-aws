pipeline {
    agent none           

    environment {
        AWS_REGION   = 'us-east-1'
        STACK_NAME   = 'todo-list-aws-staging'
        S3_BUCKET    = 'aws-sam-cli-managed-default-samclisourcebucket-qmu3tffcy8al'
        REPO         = 'todo-list-aws.git'
        USUARIO_GITHUB = 'alex19bh'
        CONFIG_REPO  = 'todo-list-aws-config'
    }

    stages {

        stage('Get Code') {
            agent { label 'principal' }
            steps {
                sh 'whoami'
                sh 'hostname'
                cleanWs()
                echo 'Descargando código desde la rama develop…'
                git branch: 'develop',
                    url: "https://github.com/${USUARIO_GITHUB}/${REPO}"

                script {
                    echo "Descargando samconfig.toml desde rama staging de repo ${CONFIG_REPO}"
                    sh """
                        git init config
                        cd config
                        git remote add origin https://github.com/${USUARIO_GITHUB}/${CONFIG_REPO}.git
                        git config core.sparseCheckout true
                        echo "samconfig.toml" > .git/info/sparse-checkout
                        git pull origin staging
                        cd ..
                        mv config/samconfig.toml ./samconfig.toml
                        rm -rf config
                    """
                }

                stash name: 'fuente', includes: '**/*'
            }
        }

        stage('Static Test') {
            agent { label 'segundonodoubuntu' }
            steps {
                sh 'whoami'
                sh 'hostname'
                cleanWs()
                unstash 'fuente'
                sh '''
                    flake8 src > flake8-report.txt || true
                    bandit -r src -f custom -o bandit-report.out --msg-template "{abspath}:{line}: [{test_id}] {msg}" || true
                '''
                recordIssues tools: [
                    flake8(pattern: 'flake8-report.txt'),
                    pyLint(name: 'Bandit', pattern: 'bandit-report.out')
                ]
                stash name: 'fuente', includes: '**/*'
            }
        }

        stage('Deploy') {
            agent { label 'principal' }
            steps {
                sh 'whoami'
                sh 'hostname'
                cleanWs()
                unstash 'fuente'
                sh '''
                    sam build
                    sam deploy --config-file samconfig.toml --config-env staging --no-fail-on-empty-changeset
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
                sh 'whoami'
                sh 'hostname'
                cleanWs()
                unstash 'fuente'
                sh '''
                    echo "Usando BASE_URL=$BASE_URL"
                    export BASE_URL
                    sleep 10
                    python3 -m pytest test/integration/todoApiTest.py
                '''
            }
        }

stage('Promote') {
    when { expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' } }
    agent { label 'principal' }
    steps {
        sh 'whoami'
        sh 'hostname'
        cleanWs()  // Limpiar workspace antes de empezar
        // Clonar repo limpio
        git branch: 'develop',
            url: "https://github.com/${USUARIO_GITHUB}/${REPO}"

        withCredentials([usernamePassword(
            credentialsId: 'github-token',
            usernameVariable: 'GIT_USER',
            passwordVariable: 'GIT_TOKEN')]) {
            sh '''
                REMOTE_URL="https://${GIT_USER}:${GIT_TOKEN}@github.com/${USUARIO_GITHUB}/${REPO}"
                git checkout master || git checkout -b master origin/master
                git merge origin/develop
                git push "$REMOTE_URL" master
            '''
        }
    }
}

    }
}
