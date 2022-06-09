pipeline {
    agent { label 'main' }

    stages {
        stage('Make or check dev') {
            steps {
                cleanWs()
                sh 'git clone git@github.com:millcons/terraform-cicd-task.git'
                dir('terraform-cicd-task/dev/') {
                    sh 'terraform init'
                    sh 'terraform apply -auto-approve'
                }
            }
        }
        stage('Build on dev') {
            steps {
                node('dev') {
                    dir('spring-petclinic') {
                        deleteDir()
                    }
                    //sh 'git clone https://github.com/millcons/spring-petclinic.git'
                    dir('spring-petclinic') {
                        checkout([$class: 'GitSCM', branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[credentialsId: 'aws-pem', url: 'https://github.com/millcons/spring-petclinic.git']]])
                        sh 'mvn package'
                    }
                }
            }
        }    
        stage('Deliver artifact to s3') {
            steps {
                node('dev') {
                    withCredentials([[
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: "awsdeployer",
                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                    ]]) {
                        sh 'aws s3 cp /home/ubuntu/workspace/DeployPipeline/spring-petclinic/target/spring-petclinic-2.7.0-SNAPSHOT.jar s3://artifactkeeper/'
                    }
                }
            }
        }    
        stage('Deploy to prod?') {
            steps {
                echo 'input message:"Deploy to prod?"'
                //input message:"Deploy to prod?"
            }
        }
        stage('Make or check prod') {
            steps {
                    dir('terraform-cicd-task/prod/') {
                        sh 'terraform init'
                        sh 'terraform apply -auto-approve'
                }
            }
        }
        stage('Copy artifact from s3') {
            steps {
                node('prod') {
                    withCredentials([[
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: "awsdeployer",
                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                    ]]) {
                        sh 'aws s3 cp s3://artifactkeeper/spring-petclinic-2.7.0-SNAPSHOT.jar /home/ubuntu/'
                    }
                }
            }
        } 
        stage('Start app') {
            steps {
                node('prod') {
                    //sh 'sudo chown :ubuntu /var/lib/tomcat9/webapps'
                    sh 'sudo systemctl stop tomcat9'
                    sh 'mkdir -p /home/ubuntu/webapp'
                    sh 'rm -rf /home/ubuntu/webapp/spring-petclinic.jar'
                    sh 'cp /home/ubuntu/spring-petclinic-2.7.0-SNAPSHOT.jar /home/ubuntu/webapp/spring-petclinic.jar'
                    withEnv(['JENKINS_NODE_COOKIE=do_not_kill']) {
                    sh 'java -jar /home/ubuntu/webapp/spring-petclinic.jar &'
                    }
                    echo 'Done!'
                    //sh 'java -jar -Dspring.profiles.active=mysql /home/ubuntu/spring-petclinic-2.7.0-SNAPSHOT.jar'
                }
            }
        }
        stage('finish') {
            steps {
                echo "finish"
                emailext body: "Pipeline ${JOB_NAME} built ${currentBuild.currentResult}", subject: 'job.notification for build ${BUILD_ID}', to: 'konstantin.in.ua@gmail.com'
            }
        }
    }
}