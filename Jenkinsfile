pipeline {
    agent any
    environment {
        AWS_ACCOUNT_ID="520261045384"
        AWS_DEFAULT_REGION="us-west-2"
        CLUSTER_NAME="nodejs1"
        DESIRED_COUNT="1"
        IMAGE_REPO_NAME="nodejs_project"
        TASK_DEFINITION_NAME="nj_td"
        //Do not edit the variable IMAGE_TAG. It uses the Jenkins job build ID as a tag for the new image.
        IMAGE_TAG="${env.BUILD_ID}"
        //Do not edit REPOSITORY_URI.
        REPOSITORY_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}"
        SUBNETS = "[subnet-0a2b1d7a381e7833e]"
        SECURITYGROUPS = "[sg-06e2e7509ad6231d7]"
        SERVICE_NAME = "nodejs-svc"
        VPC_ID ="vpc-07ee561ec3aba1ec7"
        // For Auto Scaling groups
        ASG_NAME = "test-asg"
        MIN_CAPACITY = "1"
        MAX_CAPACITY = "2"
        lbArn = ""
        tgArn = ""
        asgArn = ""
    }
   
    stages {
        stage('checkout') {
            steps {
                git url: 'https://github.com/G-Dhanusha/Nodejs_project.git',
                    branch: 'dev'
            }
        }
 
        // Building Docker image
        stage('Building image') {
            steps{
                script {
                    dockerImage = docker.build ("${IMAGE_REPO_NAME}:${IMAGE_TAG}")
                }
            }
        }
        // Uploading Docker image into AWS ECR
        stage('Releasing') {
            steps{  
                script {
                    sh "aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com"
                    sh "docker tag ${IMAGE_REPO_NAME}:${IMAGE_TAG} ${REPOSITORY_URI}:${IMAGE_TAG}"
                    sh "docker push ${REPOSITORY_URI}:${IMAGE_TAG}"
                    }
                }
            }
        // Creating an Application Load Balancer
        stage('Create Load Balancer') {
            steps {
                script {
                    def lbOutput = sh(script: """
                        aws elbv2 create-load-balancer \
                            --name Nodejs-lb \
                            --subnets subnet-0a2b1d7a381e7833e \
                            --security-groups sg-06e2e7509ad6231d7 \
                            --type application \
                            --ip-address-type ipv4 \
                            --scheme internet-facing \
                            --region ${AWS_DEFAULT_REGION} \
                            --output json
                    """, returnStdout: true).trim()
 
                    // Use jq to extract the Load Balancer ARN from the JSON output
                    lbArn = sh(script: "echo '${lbOutput}' | jq -r '.LoadBalancers[0].LoadBalancerArn'", returnStdout: true).trim()
                   
                    // Print the ARN for verification
                    println "Load Balancer ARN: ${lbArn}"
                }
            }
        }
        // Creating a target group
        stage('Create targetgroup') {
            steps {
                script {
                    def tgOutput = sh(script: """
                        aws elbv2 create-target-group \
                            --name asg-apptg \
                            --protocol HTTP \
                            --port 80 \
                            --vpc-id ${VPC_ID} \
                            --health-check-protocol HTTP \
                            --health-check-path / \
                            --target-type instance \
                            --ip-address-type ipv4 \
                            --region ${AWS_DEFAULT_REGION} \
                            --output json
                    """, returnStdout: true).trim()
 
                    // Use jq to extract the Target Group ARN from the JSON output
                    tgArn = sh(script: "echo '${tgOutput}' | jq -r '.TargetGroups[0].TargetGroupArn'", returnStdout: true).trim()
 
                    // Print the ARN for verification
                    println "Target Group ARN: ${tgArn}"
                }
            }
        }
       
        // Creating a listener
        stage('Create listener') {
            steps {
                script {
                    sh """
                        aws elbv2 create-listener --load-balancer-arn ${lbArn} --protocol HTTP --port 80 --default-actions Type=forward,TargetGroupArn=${tgArn} --region ${AWS_DEFAULT_REGION}
                    """
                }
            }
        }
       
        // Creating autoscaling group
        stage('Creating autoscaling group') {
            steps {
                script {
                    def asgExists = sh(script: """
                        aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names ${ASG_NAME} --region ${AWS_DEFAULT_REGION} --query "length(AutoScalingGroups)"
                    """, returnStdout: true).trim()
 
                    if (asgExists.toInteger() == 0) {
                        // ASG doesn't exist, so create it
                        def sgOutput = sh(script: """
                            aws autoscaling create-auto-scaling-group \
                                --auto-scaling-group-name ${ASG_NAME} \
                                --launch-template Version=1,LaunchTemplateId=lt-05f22abfcfbedacb3 \
                                --health-check-type EC2 \
                                --health-check-grace-period 300 \
                                --desired-capacity ${DESIRED_COUNT} \
                                --min-size ${MIN_CAPACITY} \
                                --max-size ${MAX_CAPACITY} \
                                --vpc-zone-identifier "${SUBNETS}"
                        """, returnStdout: true).trim()
                       
                        // Extract ASG ARN from output
                        asgArn = sh(script: "echo '${sgOutput}' | jq -r '.AutoScalingGroups[0].AutoScalingGroupARN'", returnStdout: true).trim()
                       
                        // Print ASG ARN
                        println "Auto Scaling Group ARN: ${asgArn}"
                    } else {
                        // ASG already exists, so update it
                        echo "Auto Scaling Group '${ASG_NAME}' already exists. Updating..."
                        def updateOutput = sh(script: """
                            aws autoscaling update-auto-scaling-group \
                                --auto-scaling-group-name ${ASG_NAME} \
                                --launch-template Version=1,LaunchTemplateId=lt-0ab66584a121e451c \
                                --desired-capacity ${DESIRED_COUNT} \
                                --min-size ${MIN_CAPACITY} \
                                --max-size ${MAX_CAPACITY} \
                        """, returnStdout: true).trim()
                       
                        // Extract ASG ARN from output
                        asgArn = sh(script: "echo '${updateOutput}' | jq -r '.AutoScalingGroups[0].AutoScalingGroupARN'", returnStdout: true).trim()
                       
                        // Print ASG ARN
                        println "Auto Scaling Group ARN: ${asgArn}"
                    }
                }
            }
        }
 
        stage('Create Capacity Provider') {
            steps {
                script {
                    sh "aws ecs create-capacity-provider --name ecs_asg --auto-scaling-group-provider autoScalingGroupArn=${asgArn}"
                }
            }
        }
 
        // Cluster creation
        stage('Create ECS Cluster') {
            steps {
                sh "aws ecs create-cluster --cluster-name ${CLUSTER_NAME} --region ${AWS_DEFAULT_REGION} --capacity-providers ecs_asg "
            }
        }
        stage('Register Task Definition') {
            steps {
                script {
                    writeFile file: 'task-definition.json', text: """
                    {
                        "containerDefinitions": [
                            {
                                "name": "nodejs_container",
                                "image": "${REPOSITORY_URI}:${IMAGE_TAG}",
                                "essential": true,
                                "cpu": 256,
                                "memory": 512,
                                "portMappings": [
                                    {
                                        "containerPort": 3000,
                                        "hostPort": 3000
                                    }
                                ]
                            }
                        ],
                        "family": "${TASK_DEFINITION_NAME}",
                        "taskRoleArn": "arn:aws:iam::520261045384:role/ecsTaskExecutionRole",
                        "executionRoleArn": "arn:aws:iam::520261045384:role/ecsTaskExecutionRole",
                        "requiresCompatibilities": ["FARGATE"],
                        "networkMode": "awsvpc",
                        "cpu": "256",
                        "memory": "512"
                    }
                    """
                    sh "aws ecs register-task-definition --cli-input-json file://task-definition.json --region ${AWS_DEFAULT_REGION} "
                   
                }
            }
        }
       
        stage('Deploy') {
            steps {
                script {
                        int count = sh(script: """
                            aws ecs describe-services --cluster ${CLUSTER_NAME} --services ${SERVICE_NAME} | jq '.services | length'
                        """,
                        returnStdout: true).trim()
 
                        echo "service count: $count"
 
                        if (count > 0) {
                            //ECS service exists: update
                            echo "Updating ECS service $ecs_service..."
                            sh(script: """
                                aws ecs update-service \
                                    --cluster ${CLUSTER_NAME} \
                                    --service ${SERVICE_NAME} \
                                    --task-definition ${TASK_DEFINITION_NAME} \
                            """)
                        }
                        else {
                            //ECS service does not exist: create new
                            echo "Creating new ECS service $ecs_service..."
                   
                            sh(script: """
                                aws ecs create-service \
                                    --cluster ${CLUSTER_NAME} \
                                    --task-definition ${TASK_DEFINITION_NAME} \
                                    --service-name ${SERVICE_NAME} \
                                    --desired-count 1 \
                                    --launch-type EC2 \
                                    --scheduling-strategy REPLICA \
                                    --load-balancers "targetGroupArn=${tgArn},containerName=nodejs_container,containerPort=3000" \
                                    --deployment-configuration "maximumPercent=200,minimumHealthyPercent=100"
                            """)
                        }                
                }  
 
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