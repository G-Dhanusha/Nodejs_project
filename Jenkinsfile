pipeline {
    agent any
    environment {
        AWS_ACCOUNT_ID="533267223585"
        AWS_DEFAULT_REGION="us-west-1"
        CLUSTER_NAME="nodejs_test"
        TASK_DEFINITION_NAME="nodejs_task"
        DESIRED_COUNT="1"
        IMAGE_REPO_NAME="nodejs_project"
        //Do not edit the variable IMAGE_TAG. It uses the Jenkins job build ID as a tag for the new image.
        IMAGE_TAG="${env.BUILD_ID}"
        //Do not edit REPOSITORY_URI.
        REPOSITORY_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}"
        registryCredential = "aws"
        SERVICE_NAME = "nodejs-svc"
        VPC_ID ="vpc-02a0aad6b2049d61f"
        SUBNETS = "subnet-0f2229e52f410cfce,subnet-02bd443789219bf11"
        SECURITYGROUPS = "sg-06db6a49e9854060a"
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
       
        stage('Create Load Balancer') {
            steps {
                script {
                    // Creating an Application Load Balancer
                    def lbOutput = sh(script: """
                        aws elbv2 create-load-balancer \
                            --name Nodejs-lb \
                            --subnets subnet-0f2229e52f410cfce subnet-02bd443789219bf11 \
                            --security-groups ${SECURITYGROUPS} \
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
 
        stage('Create targetgroup') {
            steps {
                script {
                    // Creating a target group
                    def tgOutput = sh(script: """
                        aws elbv2 create-target-group \
                            --name app-tg \
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
 
        stage('Create listener') {
            steps {
                script {
                    // Creating a listener
                    sh """
                        aws elbv2 create-listener \
                        --load-balancer-arn ${lbArn} \
                        --protocol HTTP \
                        --port 80 \
                        --default-actions Type=forward,TargetGroupArn=${tgArn} \
                        --region ${AWS_DEFAULT_REGION}
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
                        echo "Creating Auto Scaling Group..."
                        def sgOutput = sh(script: """
                            aws autoscaling create-auto-scaling-group \
                                --auto-scaling-group-name ${ASG_NAME} \
                                --launch-template Version=1,LaunchTemplateId=lt-0fac26265b0845152 \
                                --health-check-type EC2 \
                                --health-check-grace-period 300 \
                                --desired-capacity ${DESIRED_COUNT} \
                                --min-size ${MIN_CAPACITY} \
                                --max-size ${MAX_CAPACITY} \
                                --vpc-zone-identifier "subnet-0f2229e52f410cfce,subnet-02bd443789219bf11"
                        """, returnStdout: true).trim()
 
                        echo "Auto Scaling Group creation output: ${sgOutput}"
 
                        // Extract ASG ARN from output
                        asgArn = sh(script: "echo '${sgOutput}' | jq -r '.AutoScalingGroups[0].AutoScalingGroupARN'", returnStdout: true).trim()
 
                        // Print ASG ARN
                        println "Auto Scaling Group ARN: ${asgArn}"
                    } else {
                        // ASG already exists
                        echo "Auto Scaling Group '${ASG_NAME}' already exists. Skipping creation..."
                        def asgPresent = sh(script: """
                            aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names ${ASG_NAME} --region ${AWS_DEFAULT_REGION} --query "AutoScalingGroups[0].AutoScalingGroupARN" --output text
                            """, returnStdout: true).trim()
                        asgArn = asgPresent
                        println "Auto Scaling Group ARN: ${asgArn}"
                    }
                }
            }
        }
        // Create Capacity Provider
        stage('Create Capacity Provider') {
            when {
                expression { asgArn != "" }
            }
            steps {
                script {
                    // Check if the capacity provider already exists
                    def capacityProviderExists = sh(script: """
                        aws ecs describe-capacity-providers --capacity-providers ecs_cp --region ${AWS_DEFAULT_REGION} --query "length(capacityProviders)"
                    """, returnStdout: true).trim()
 
                    if (capacityProviderExists.toInteger() == 0) {
                        // Capacity provider doesn't exist, create it
                        sh "aws ecs create-capacity-provider --name ecs_cp --auto-scaling-group-provider autoScalingGroupArn=${asgArn} --region ${AWS_DEFAULT_REGION}"
                    } else {
                        echo "capacity provider already exists"
                    }
                }
            }
        }
        // Cluster creation
        stage('Create ECS Cluster') {
            steps {
                sh "aws ecs create-cluster \
                --cluster-name ${CLUSTER_NAME} \
                 --capacity-providers ecs_cp \
                 --default-capacity-provider-strategy capacityProvider=ecs_cp,weight=1 \
                --region ${AWS_DEFAULT_REGION}"
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
                                        "awslogs-group": "/ecs/${TASK_DEFINITION_NAME}",
                                        "awslogs-region": "${AWS_DEFAULT_REGION}",
                                        "awslogs-stream-prefix": "ecs"
                                    },
                                    "secretOptions": []
                                },
                                "systemControls": []
                            }
                        ],
                        "taskRoleArn": "arn:aws:iam::${AWS_ACCOUNT_ID}:role/AmazonEC2@EKS",
                        "executionRoleArn": "arn:aws:iam::${AWS_ACCOUNT_ID}:role/AmazonEC2@EKS",
                        "networkMode": "awsvpc",
                        "requiresCompatibilities": [
                            "EC2"
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
        stage('Describe and Create ECS Service') {
            steps {
                script {
                    def serviceDescription = sh(script: "aws ecs describe-services --cluster ${CLUSTER_NAME} --service ${SERVICE_NAME}", returnStdout: true).trim()
                    if (serviceDescription.contains('does not exist')) {
                        echo "ECS service does not exist, creating a new one..."
                        sh """
                            aws ecs create-service \
                                --cluster ${CLUSTER_NAME} \
                                --service-name ${SERVICE_NAME} \
                                --task-definition ${TASK_DEFINITION_NAME} \
                                --launch-type EC2 \
                                --scheduling-strategy REPLICA \
                                --load-balancers targetGroupArn=${tgArn},containerName="nodejs_container",containerPort=3000 \
                                --desired-count 1 \
                                --capacity-providers EC2 ${ASG_NAME}
                        """
                    } else {
                        echo "ECS service already exists, updating capacity providers..."
                        sh """
                            aws ecs update-service \
                                --cluster ${CLUSTER_NAME} \
                                --service ${SERVICE_NAME} \
                                --capacity-providers FARGATE ${ASG_NAME}
                        """
                    }
                }
            }
        }
    //             sh """
    //                 aws application-autoscaling register-scalable-target \
    //                     --service-namespace ecs \
    //                     --scalable-dimension ecs:service:DesiredCount \
    //                     --resource-id service/${CLUSTER_NAME}/${SERVICE_NAME}\
    //                     --min-capacity ${MIN_CAPACITY} \
    //                     --max-capacity ${MAX_CAPACITY} \
    //                     --region ${AWS_DEFAULT_REGION}
    //                 """                
    //         }
    //     }
    // }
 
        // Clear local image registry. Note that all the data that was used to build the image is being cleared.
        // For different use cases, one may not want to clear all this data so it doesn't have to be pulled again for each build.
   post {
       always {
       sh 'docker system prune -a -f'
       
     }
   }
 }
}