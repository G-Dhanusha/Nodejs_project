pipeline {
    agent any
    environment {
        AWS_ACCOUNT_ID="520261045384"
        AWS_DEFAULT_REGION="us-east-2"
	    CLUSTER_NAME="nodejs_test"
	    TASK_DEFINITION_NAME="nodejs_task"
	    DESIRED_COUNT="1"
        IMAGE_REPO_NAME="nodejs_project_1"
        //Do not edit the variable IMAGE_TAG. It uses the Jenkins job build ID as a tag for the new image.
        IMAGE_TAG="${env.BUILD_ID}"
        //Do not edit REPOSITORY_URI.
        REPOSITORY_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}"
	    registryCredential = "aws_credentials"
        SUBNETS = "[subnet-062813342cba59761,subnet-025c57b2325a376f0]"
        SECURITYGROUPS = "[	sg-06e2e7509ad6231d7]"
    
    }
   
    stages {
        stage('checkout') {
            steps {
                git url: 'https://github.com/G-Dhanusha/Nodejs_project.git',
                    branch: 'main'
            }
        }

        // Building Docker image
        stage('Building image') {
            steps{
                script {
                    dockerImage = docker.build "${IMAGE_REPO_NAME}:${IMAGE_TAG}"
                }
            }
        }
        // Uploading Docker image into AWS ECR
        stage('Releasing') {
            steps{  
                script {
                    docker.withRegistry("https://" + REPOSITORY_URI, "ecr:${AWS_DEFAULT_REGION}:" + registryCredential) {
                                dockerImage.push()
                    }
                }
            }
        }
        // Cluster creation
        stage('Create ECS Cluster') {
            steps {
                sh "aws ecs create-cluster --cluster-name ${CLUSTER_NAME} --region ${AWS_DEFAULT_REGION}"
            }
        }
        // Creating the Task Definition
        stage('Register Task Definition') {
            steps {
                script {
                    writeFile file: 'task-definition.json', text: """
                    {
                        "family": "${TASK_DEFINITION_NAME}",
                        "containerDefinitions": [
                            {
                                "name": "nodejs_container",
                                "image": "${REPOSITORY_URI}:${IMAGE_TAG}",
                                "cpu": 0,
                                "portMappings": [
                                    {
                                        "name": "nodejs_port",
                                        "containerPort": 3000,
                                        "hostPort": 3000,
                                        "protocol": "tcp",
                                        "appProtocol": "http"
                                    }
                                ],
                                "essential": true,
                                "environment": [],
                                "environmentFiles": [],
                                "mountPoints": [],
                                "volumesFrom": [],
                                "ulimits": [],
                                "logConfiguration": {
                                    "logDriver": "awslogs",
                                    "options": {
                                        "awslogs-create-group": "true",
                                        "awslogs-group": "/ecs/nodejs_task",
                                        "awslogs-region": "us-west-2",
                                        "awslogs-stream-prefix": "ecs"
                                    },
                                    "secretOptions": []
                                },
                                "systemControls": []
                            }
                        ],
                        "taskRoleArn": "arn:aws:iam::520261045384:role/ECS_Role",
                        "executionRoleArn": "arn:aws:iam::520261045384:role/ECS_Role",
                        "networkMode": "awsvpc",
                        "requiresCompatibilities": [
                            "FARGATE"
                        ],
                        "cpu": "1024",
                        "memory": "3072",
                        "runtimePlatform": {
                            "cpuArchitecture": "X86_64",
                            "operatingSystemFamily": "LINUX"
                        }
                    }
                    """
                    sh "aws ecs register-task-definition --cli-input-json file://task-definition.json --region ${AWS_DEFAULT_REGION}"
                }
            }
        }
        // Run the task
        stage('Run Task in ECS') {
            steps {
                sh "aws ecs run-task --cluster ${CLUSTER_NAME} --task-definition ${TASK_DEFINITION_NAME} --region ${AWS_DEFAULT_REGION} --launch-type FARGATE --count ${DESIRED_COUNT} --network-configuration awsvpcConfiguration={subnets=${SUBNETS},securityGroups=${SECURITYGROUPS},assignPublicIp=ENABLED} "
            }
        }
    }
        // Clear local image registry. Note that all the data that was used to build the image is being cleared.
        // For different use cases, one may not want to clear all this data so it doesn't have to be pulled again for each build.
    post {
       always {
       sh 'docker system prune -a -f'
     }
    }
 }