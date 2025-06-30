pipeline {
    }

    stages {

        stage('Get Code') {
            agent { label 'principal' }
            steps {
                sh 'whoami'
                sh 'hostname'
                cleanWs()
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

    }
}

    }
}
