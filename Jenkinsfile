pipeline {
    agent any

    environment {
        REGISTRY_URL = "wimetrixcregistery.azurecr.io"
        REGISTRY_CREDENTIALS_ID = "acr-credentials"
        //IMAGE_NAME = "wimetrixcregistery/Nishat-backend-report"
        GIT_CREDENTIALS_ID = "GithubCredentials"
        SOURCE_REPO = "github.com/WiMetrixDev/sooperwizer.git"
        WORKSPACE_DIR = "/home/jenkins/deployment-package/wimetrix/Nishat-backend-report"
        DEPLOYMENT_YAML = "/home/jenkins/deployment-package/wimetrix/Nishat-backend-report/deployments/deployment.yaml"
        GITHUB_REPO = 'github.com/WiMetrixDev/dep-Nishat-backend-report.git'

    }

    stages {

            stage('Set Environment Variables') {
            steps {
                script
                {
                    
                    if (params.DEP_BRANCH == 'main') {
                        env.IMAGE_NAME = "wimetrixcregistery/Nishat-backend-report"
                    } 
                    
                    else if (params.DEP_BRANCH == 'qa') {
                        env.IMAGE_NAME = "wimetrixcregistery/Nishat-backend-reportqa"
                    } 
                    
                    else {
                        error "Branch not supported for deployment."
                    }
                    
                    echo "Using environment configuration for branch: ${env.DEP_BRANCH}"
                    echo "IMAGE_NAME: ${env.IMAGE_NAME}"
                    echo "WORKSPACE_DIR: ${env.WORKSPACE_DIR}"
                    echo "DEPLOYMENT_YAML: ${env.DEPLOYMENT_YAML}"
                }
            }
        }





        stage('Checkout') {
        steps {
            // Checkout first repo
            checkout([$class: 'GitSCM',
                    branches: [[name: 'remotes/origin/$DEP_BRANCH']],
                    doGenerateSubmoduleConfigurations: false,
                    extensions: [
                        [$class: 'WipeWorkspace'],
                        [$class: 'RelativeTargetDirectory', relativeTargetDir: '/home/jenkins/deployment-package/wimetrix/Nishat-backend-report']
                    ],
                    submoduleCfg: [],
                    userRemoteConfigs: [[
                        credentialsId: 'GithubCredentials',
                        url: 'https://github.com/WiMetrixDev/dep-Nishat-backend-report.git'
                    ]]
            ])

            // Checkout second repo
            checkout([$class: 'GitSCM',
                    branches: [[name: 'remotes/origin/$SOURCE_BRANCH']], 
                    doGenerateSubmoduleConfigurations: false,
                    extensions: [
                        [$class: 'WipeWorkspace'],
                        [$class: 'RelativeTargetDirectory', relativeTargetDir: '/home/jenkins/deployment-package/wimetrix/Nishat-backend-report/source']
                    ],
                    submoduleCfg: [],
                    userRemoteConfigs: [[
                        credentialsId: 'GithubCredentials',
                        url: 'https://github.com/WiMetrixDev/sooperwizer.git'
                    ]]
            ])
        }
    }


        stage('Build Docker Image') {
    steps {
        script {
            def dockerfileDir = '/home/jenkins/deployment-package/wimetrix/Nishat-backend-report/'
            def sourceDir = "${dockerfileDir}source"
            withCredentials([
                        file(credentialsId: 'NISHAT_BACKEND_COMMON_ENV_PROD', variable: 'SECRET_FILE'),
                        file(credentialsId: 'NISHAT_BACKEND_REPORT_PRODUCTION', variable: 'SECRET_FILE2' )
                    ]){

                    sh label: '', script: '''
                        cp /home/jenkins/deployment-package/wimetrix/Nishat-backend-report/Dockerfile /home/jenkins/deployment-package/wimetrix/Nishat-backend-report/source/
                        cp /home/jenkins/deployment-package/wimetrix/Nishat-backend-report/.dockerignore /home/jenkins/deployment-package/wimetrix/Nishat-backend-report/source/
                        cd /home/jenkins/deployment-package/wimetrix/Nishat-backend-report/source/shared-env/backend
                        cp ${SECRET_FILE} /home/jenkins/deployment-package/wimetrix/Nishat-backend-report/source/shared-env/backend/.env.production
                        cd /home/jenkins/deployment-package/wimetrix/Nishat-backend-report/source/apps/backend-report
                        cp ${SECRET_FILE2} /home/jenkins/deployment-package/wimetrix/Nishat-backend-report/source/apps/backend-report/.env.production
                '''
                dockerImage = docker.build("${REGISTRY_URL}/${IMAGE_NAME}:${env.BUILD_ID}",
                    "-f ${sourceDir}/Dockerfile ${sourceDir}")

                sh label: '', script: '''
                rm -rf /home/jenkins/deployment-package/wimetrix/Nishat-backend-report/source/*
                ''' 
                    }                 
            }
        }
    }
  

        stage('Push Docker Image') {
            steps {
                script {
                    docker.withRegistry("https://${REGISTRY_URL}", "${REGISTRY_CREDENTIALS_ID}") {
                        dockerImage.push("${env.BUILD_ID}")
                        dockerImage.push("latest")

                    }
                }
            }
        }

        stage('Trigger GitHub Actions'){
            steps {
                script {
                    withCredentials([string(credentialsId: 'GITHUB_ACTIONS_SECRET', variable: 'GITHUB_ACTIONS')]){
                        def githubPAT = env.GITHUB_ACTIONS
                        def githubOwner = "WiMetrixDev"  
                        def githubRepo = "dep-Nishat-backend-report"  
                        def githubWorkflow = "production-deploy.yaml"  
                        def branch = "main" 

                        def apiUrl = "https://api.github.com/repos/${githubOwner}/${githubRepo}/actions/workflows/${githubWorkflow}/dispatches"

                        def payload = """
                        {
                            "ref": "${branch}"
                        }
                        """

                        echo "Triggering GitHub Actions workflow: ${githubWorkflow} for repo: ${githubRepo} on branch: ${branch}"
                        echo "API URL: ${apiUrl}"

                        def response = httpRequest(
                            url: apiUrl,
                            httpMode: 'POST',
                            contentType: 'APPLICATION_JSON',
                            customHeaders: [[name: 'Authorization', value: "Bearer ${githubPAT}"]],
                            requestBody: payload,
                            validResponseCodes: '201,204'
                        )

                        echo "GitHub API response: ${response}"
                    }
                    
                }
            }
        }


    }


    post {
        always {
            cleanWs()
        }
        success {
            script {
                // Use withCredentials to access the webhook URL from Jenkins credentials
                withCredentials([string(credentialsId: 'TEAMS_WEBHOOK_URL', variable: 'WEBHOOK_URL'),
                string(credentialsId: 'JENKINS_API_TOKEN', variable: 'JENKINS_TOKEN'),
                string(credentialsId: 'JENKINS_USER', variable: 'JENKINS_USER')
                ]
                ) {
                    def buildUrl = env.BUILD_URL // Get the Jenkins build URL
                    def logUrl = "${buildUrl}console" // Create a link to the build logs
                    def logs = "${buildUrl}consoleText"
                    def logContent = sh(script: "curl -u ${JENKINS_USER}:${JENKINS_TOKEN} -s ${buildUrl}consoleText", returnStdout: true).trim()

                    // Send notification to Teams with a link to the log file
                    office365ConnectorSend message: "Build Succeeded For $DEP_BRANCH. Check the logs \n```${logContent}``` ", 
                                           status: "Success", 
                                           webhookUrl: env.WEBHOOK_URL
                }
            }
         
        }
        failure {
            script {
                // Use withCredentials to access the webhook URL from Jenkins credentials    
                withCredentials([string(credentialsId: 'TEAMS_WEBHOOK_URL', variable: 'WEBHOOK_URL'),
                string(credentialsId: 'JENKINS_API_TOKEN', variable: 'JENKINS_TOKEN'),
                string(credentialsId: 'JENKINS_USER', variable: 'JENKINS_USER')
                ]
                ) {
                    def buildUrl = env.BUILD_URL
                    def logUrl = "${buildUrl}console"
                    def logs = "${buildUrl}consoleText"
                    def logContent = sh(script: "curl -u ${JENKINS_USER}:${JENKINS_TOKEN} -s ${buildUrl}consoleText", returnStdout: true).trim()

                    // Send notification to Teams with a link to the log file
                    office365ConnectorSend message: "Build Failed For $DEP_BRANCH. Check the logs \n```${logContent}```", 
                                           status: "Failed", 
                                           webhookUrl: env.WEBHOOK_URL
                }
            }
        }
    }
}
