pipeline {
    agent any    
    
    environment {
        AWS_REGION = 'us-east-1'
        STACK_NAME = 'todo-list-aws'
        TEMPLATE_FILE = 'template.yaml'
        CAPABILITIES = 'CAPABILITY_IAM'
        GIT_URL = 'https://github.com/jdcr-dev/unir-todo-list-aws.git'
        GIT_TOKEN = 'GIT_HUB_TOKEN'
    }
    
    stages {
        
        stage('Get Code') {
            steps {
                git branch: 'develop', 
                    credentialsId: GIT_TOKEN,
                    url: GIT_URL
            }
        }
        
        stage('Static Test') {
            steps {
                script {
                    def FOLDER_STATIC_TEST = "src"
                    
                    // He tenido que añadir esta comprobacion por que Bandit no me devolvia error si la carpeta la escribia mal.
                    if(!fileExists(FOLDER_STATIC_TEST)) {
                        error "No existe la carpeta ${FOLDER_STATIC_TEST}"
                    }
                    
                    sh """
                        flake8 --exit-zero --format=pylint ${FOLDER_STATIC_TEST} > flake8.out
                    """
                    
                    recordIssues tools: [flake8(name: 'Flake8', pattern: 'flake8.out')], 
                        qualityGates:[
                            [threshold: 8, type: 'TOTAL', unstable: false], 
                            [threshold: 10, type:'TOTAL', unstable: false]],
                        enabledForFailure: false
                    
     
                    sh """
                        bandit -r ${FOLDER_STATIC_TEST} -f custom -o bandit.out --msg-template "{abspath}:{line}: [{test_id}] {msg}" | echo "Bandit finished"
                    """
                    
                    recordIssues tools: [pyLint(name: 'Bandit', pattern: 'bandit.out')], 
                        qualityGates:[
                            [threshold: 2, type: 'TOTAL', unstable: false], 
                            [threshold: 4, type:'TOTAL', unstable: false]],
                        enabledForFailure: false
                }
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
                                   --parameter-overrides Stage=staging \
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
                    pytest --junitxml=result-rest.xml test/integration/todoApiTest.py
                '''
                junit 'result*.xml'
            }
        }
        
        stage('Promote') {
            steps {
                sh '''
                    git remote set-url origin https://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@github.com/jdcr-dev/unir-todo-list-aws.git
                    
                    git checkout master
                    git pull origin master
                    git fetch origin master
                    
                    # || True es para evitar el exit code que generaba el comando ya que habia que resolver un conflicto
                    git merge origin/develop --no-commit --no-ff || true

                    # Volvemos al estado anterio del fichero
                    git reset HEAD Jenkinsfile

                    # Restauramos el Jenkisfile que original de la rama
                    git checkout -- Jenkinsfile

                    # Stage all other changes
                    git add .

                    # Creamos el comit del merge
                    git commit -m "Merged DEVELOP into MASTER, excluding Jenkinsfile"
                    
                    # Subimos los cambios
                    git push
                '''
            }
        }
        
    }
    
    
    post {
        always {
            cleanWs()
        }
    }
}
