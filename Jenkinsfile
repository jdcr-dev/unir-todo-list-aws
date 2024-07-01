
pipeline {
    agent any    
    
    environment {
        AWS_REGION = 'us-east-1'
        STACK_NAME = 'todo-list-aws-production'
        TEMPLATE_FILE = 'template.yaml'
        CAPABILITIES = 'CAPABILITY_IAM'
        GIT_URL = 'https://github.com/jdcr-dev/unir-todo-list-aws.git'
        GIT_TOKEN = 'GIT_HUB_TOKEN'
    }
    
    stages {
        
        stage('Get Code') {
            steps {
                git branch: 'master', 
                    credentialsId: GIT_TOKEN,
                    url: GIT_URL
            }
        }
        
        stage('Deploy') {
            steps {
                
                script {
                    sh """
                        sam validate --template-file ${TEMPLATE_FILE} --region ${AWS_REGION}
                        
                        sam build
                        
                        sam deploy --template-file ${TEMPLATE_FILE} \
                                   --stack-name ${STACK_NAME} \
                                   --region ${AWS_REGION} \
                                   --parameter-overrides Stage=production \
                                   --capabilities ${CAPABILITIES} \
                                   --no-confirm-changeset \
                                   --debug \
                                   --no-fail-on-empty-changeset
                    """
                    
                     def output_Base_URL = sh(script: """
                        aws cloudformation describe-stacks --stack-name ${STACK_NAME} --query "Stacks[0].Outputs[?OutputKey=='BaseUrlApi'].OutputValue" --output text
                    """, returnStdout: true).trim()
                    
                    echo "Raw output: ${output_Base_URL}"
                    
                    env.BASE_URL = output_Base_URL
                    
                    sh """ 
                        echo "[SO] - LA URL RECUPERADA ES: ${BASE_URL}"
                    """
                }
            }
        }
        
        stage('Rest Test') {
            steps {
                sh '''
                    export PYTHONPATH=.
                    pytest --junitxml=test/result-rest.xml test/integration/todoApiTest.py -k "test_api_listtodos or test_api_gettodo"
                '''
                junit 'test/result*.xml'
            }
        }
        
    }
    
    
    post {
        always {
            cleanWs()
        }
    }
    
}
