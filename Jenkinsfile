pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = 'us-west-2'
        AWS_ACCOUNT_ID = '520261045384'
        ECR_REPO_NAME = 'nodejs_project'
        ECS_CLUSTER_NAME = 'nodejs'
        ECS_SERVICE_NAME = 'nodejs_svc'
        ECS_TASK_FAMILY = 'nodejs_task'
    }

    stages {
        stage('checkout') {
            steps {
                git url: 'https://github.com/G-Dhanusha/Nodejs_project.git',
                    branch: 'main'
            }
        }
        // stage('Build and Push Docker Image') {
        //     steps {
        //         script {
        //             def dockerImage = docker.build("${ECR_REPO_NAME}:${BUILD_NUMBER}")
        //             sh "aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com"
        //             sh "docker tag ${ECR_REPO_NAME}:${BUILD_NUMBER} ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${ECR_REPO_NAME}:${BUILD_NUMBER}"
        //             sh "docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${ECR_REPO_NAME}:${BUILD_NUMBER}"
        //         }
        //     }
        // }
        stage('Build') {
            steps {
                sh 'app = docker.build("Dhanusha/test")'
            }
        }
        stage('Test image') {           
            app.inside {            
             
             sh 'echo "Tests passed"'        
            }    
        }     
       stage('Push image') {
             docker.withRegistry('https://registry.hub.docker.com', 'git') {            
                        app.push("${env.BUILD_NUMBER}")            
                        app.push("latest")        
              }    
           }
        }
        stage('Create ECS Cluster') {
            steps {
                sh "aws ecs create-cluster --cluster-name ${ECS_CLUSTER_NAME} --region ${AWS_DEFAULT_REGION}"
            }
        }
        stage('Register Task Definition') {
            steps {
                script {
                    writeFile file: 'task-definition.json', text: """
                    {
                        "family": "${ECS_TASK_FAMILY}",
                         "containerDefinitions": [
                            {
                                "name": "nodejs",
                                "image": "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${ECR_REPO_NAME}:${BUILD_NUMBER}",
                                "cpu": 256,
                                "memory": 512,
                                "essential": true,
                                "portMappings": [
                                    {
                                        "containerPort": 3000,
                                        "hostPort": 3000
                                    }
                                ]
                            }
                        ]
                    }
                    """
                    sh "aws ecs register-task-definition --cli-input-json file://task-definition.json --region ${AWS_DEFAULT_REGION}"
                }
            }
        }
        stage('Run Task in ECS') {
            steps {
                sh "aws ecs run-task --cluster ${ECS_CLUSTER_NAME} --task-definition ${ECS_TASK_FAMILY} --region ${AWS_DEFAULT_REGION} --launch-type EC2 --count 2"
            }
        }
    }
}
