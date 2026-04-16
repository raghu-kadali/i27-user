
// this is user page 
pipeline {
    agent {
        label 'java-slave'
    }

    tools {
        maven 'maven-3.8.9'
        jdk 'JDK-21'
    }

    environment {
        APPLICATION_NAME = 'user'
        // SONAR_HOST_URL = "http://35.188.126.241:9000"
        // SONAR_LOGIN_TOKEN = credentials('raghu_sonar_creds')
        POM_VERSION = readMavenPom().getVersion()
        POM_PACKAGING = readMavenPom().getPackaging()
        DOCKER_HUB = "docker.io/dockerhubraghu"
        DOCKER_CREDENTIALS = credentials('raghu_dockerhub_creds')
    }

   // parametes: used to tale imnput
   
    parameters {
        choice(name: 'build_only', // it creates dropdown to user in jenkins ui build parameters
              choices: ['yes', 'no'], description: 'Build only') //first write 'no' takes default value
        // choice(name: 'SonarQube_Analysis', 
        //       choices: ['yes', 'no'], description: 'Perform SonarQube analysis')
        choice(name: 'docker_build_and_push', 
             choices: ['yes', 'no'], description: 'Build and push Docker image')
        choice(name: 'deploy_to_dev', 
             choices: ['yes', 'no'], description: 'Deploy to dev environment')  
        choice(name: 'deploy_to_test', 
             choices: ['no', 'yes'], description: 'Deploy to test environment') 
        choice(name: 'deploy_to_stage', 
             choices: ['no', 'yes'], description: 'Deploy to stage environment')
        choice(name: 'deploy_to_prod',
             choices: ['no', 'yes'], description: 'Deploy to prod environment')


    }

    stages {
        stage('Build') {
           //writing when condition for each satge level only by use parameter input
           when {
                anyOf { // if any  one of the condition is true then only stage will execute
                    expression { params.build_only == 'yes' }
                    expression { params.docker_build_and_push == 'yes' }
                   
                }

           }
            steps { // if build application we need to call method : that method build steps available.
                script {
                    buildapp().call()
                }
            }
        }

        // stage('SonarQube Analysis') {
        //     when {
        //         anyOf { 
        //             expression { params.SonarQube_Analysis == 'yes' }
        //         }
        //     }
        //     steps {
        //         echo "*** Starting SonarQube analysis"
        //         withSonarQubeEnv('SonarQubeServer') {
        //             sh """
        //                 mvn clean verify sonar:sonar \
        //                     -Dsonar.projectKey=i27-eureka \
        //                     -Dsonar.host.url=${env.SONAR_HOST_URL} \
        //                     -Dsonar.login=${env.SONAR_LOGIN_TOKEN}
        //             """
        //         }
        //     }
        //     post {
        //         always {
        //             timeout(time: 1, unit: 'HOURS') {
        //                 waitForQualityGate abortPipeline: true 
        //             }
        //         }
        //     }
        // }

        // stage('formatBuild') {
        //     steps {
        //         echo "*** Formatting code using Spotless"
        //     }
        // }

        stage('Docker Build and Push') {
                when {
                    anyOf { //if only push do ok
    
                        expression { params.docker_build_and_push == 'yes' }
                    }
                }       
            steps {
                script {
                    dockerBuildandPush().call()
                }
            }
        }
// ---------------------------------------------------------------
        stage('Deploy to dev env') {
            when {
                anyOf {
                        expression { params.deploy_to_dev == 'yes' }
                }
            }
            steps {
                script {
                    buildapp().call()
                    imagevalidation().call()
                    dockerDeploy('dev',5232).call()
                }
                
            }
        }
//---------------------------------------------------------------
        stage('Deploy to test env') { 
            when {
                anyOf {
                        expression { params.deploy_to_test == 'yes' }
                }
            }
            steps {
               script {
                    buildapp().call()
                    imagevalidation().call()
                    dockerDeploy('test',6232).call()
                }
            }
        }
//---------------------------------------------------------------
       stage('Deploy to stage env') {
            when {
                anyOf {
                        expression { params.deploy_to_stage == 'yes' }
                }

                anyOf {
                    branch 'release/*'
                      tag pattern: "v\\d{1,2}\\.\\d{1,2}\\.\\d{1,2}", comparator: "REGEXP"
                }
            }
            steps {
                script {
                    //image validation
                    buildapp().call()  
                    imagevalidation().call() 
                    dockerDeploy('stage',7232).call()
                 }
            }
        }
//---------------------------------------------------------------
        stage('Deploy to prod') { 
            // approvals neede?
            //brnach and tag condition
            //we also wrote sometime timeout with in 5 min approve and deploy other wise stop.
            when {
                anyOf {
                        expression { params.deploy_to_prod == 'yes' }
                }
                anyOf {
                      tag pattern: "v\\d{1,2}\\.\\d{1,2}\\.\\d{1,2}", comparator: "REGEXP"
                }
            }
            steps { //omly few people push by input mechanism implemented
                timeout(time: 1800, unit: 'SECONDS') {
                       input message: 'Deploy to production?', ok: 'Deploy', submitter: 'raghu, madhu'
                }
                script {
                     
                      dockerDeploy('prod',8232).call()
                 } 
            }
        }
        
    }
}




// ------------------------***-these actual methods we write here to call in above stages to implement the every stage in pipeline simply and multiple tiems clal these methods******* ---------------------------------------------------------------------------

def buildapp(){
    return {
        echo "*** Building ${env.APPLICATION_NAME} application"
        sh "mvn clean package -DskipTests"
    }
}

    
// we can also simply write docker build and push just mention metond and call
def dockerBuildandPush() {
    return {
        echo "*** Building Docker image and pushing to registry"
        sh "cp target/i27-${env.APPLICATION_NAME}-${env.POM_VERSION}.${env.POM_PACKAGING} ./.cicd"
        // these line explination under doubts section
        sh "docker build --no-cache --build-arg JAR_SOURCE=i27-${env.APPLICATION_NAME}-${env.POM_VERSION}.${env.POM_PACKAGING} -t ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:$GIT_COMMIT ./.cicd"
        echo "*** logging into docker registry ***"
        //logins not hardcodesStored safely in Jenkins Credentials Manager
        sh "docker login -u ${DOCKER_CREDENTIALS_USR} -p ${DOCKER_CREDENTIALS_PSW}"
        echo "*** pushing docker image to registry***"
        sh "docker push ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:$GIT_COMMIT"
    }
}





// define method
//envDeploy,port is a variable
def dockerDeploy(envDeploy,port) {
    return {
        echo "*** Deploying Docker image to  environment"
                // fisrt you connect the dev serever using these withcredentials :, then stop the container if exist, remove the container if exist, then create and run the container
                //  how to secure docker credentilas using withcreds block
                withCredentials([usernamePassword(credentialsId: 'dev_madhu_creds', passwordVariable: 'PASSWORD', usernameVariable: 'USERNAME')]) {
                  
                   script {
                    try {
                    // stop the container
                    // vmipaddress is pvt ip placed in manage jenkins syyste environment varbles name and pvt ip
                   sh "sshpass -p $PASSWORD -v ssh -o StrictHostKeyChecking=no $USERNAME@$docker_vm_ip_address \"docker stop ${env.APPLICATION_NAME}-${envDeploy}\""
                   // remove the container
                   sh "sshpass -p $PASSWORD -v ssh -o StrictHostKeyChecking=no $USERNAME@$docker_vm_ip_address \"docker rm ${env.APPLICATION_NAME}-${envDeploy}\""
                    } 
                    catch (error) {
                        echo "Container ${env.APPLICATION_NAME}-${envDeploy} does not exist, skipping stop and remove steps."
                    }
                    // craete and run the container
                   sh "sshpass -p $PASSWORD -v ssh -o StrictHostKeyChecking=no $USERNAME@$docker_vm_ip_address \"docker run --name ${env.APPLICATION_NAME}-${envDeploy} -p $port:8132 -d ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:$GIT_COMMIT\""
                   }

                }
    }
}

def imagevalidation() {
    return {
        println ("*** Validating Docker image in registry ***")
        try {
            sh "docker pull ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:$GIT_COMMIT"
            echo "*** Docker image validation successful ***"
        } catch (error) { 
            println ( "*****docker image not availble in registry, so we create and push the image to registry***")
            buildapp().call()  //OK BUILD APP FIRST 
            dockerBuildandPush().call() // THEN CALL THE DOCKER BUILD and push method  ok
        }

    }
}


















// yeppudaina sytntax wrong unte munde fail avutundo





















// pipeline {
//     environment {
//         Application _Name = 'eureka'
//        POM_VERSION = readMavenPom().getVersion() //read pom and fetch the version that stores in one vatrible
//        POM_PACKAGING =readMavenPom().getPackaging() //read pom and fetch the packaging that stores in one vatrible
//     }
//     stages {
//         stage('FormatBuild') {
//             // existsing i27-eureka-0.0.1-SNAPSHOT.jar
//             // Destination: i27-eureka-buildnumber-brachname.jar
//             steps {
//                 echo " testing existing jar is i27-${env.APPLICATION_NAME}-${env.POM_VERSION}.${env.POM_PACKAGING}"
//                 echo "testing jar destination is i27-${env.APPLICATION_NAME}-${env.BUILD_NUMBER}-${env.BRANCH_NAME}.${env.POM_PACKAGING}"
              
//             }
//         }
//     }
// }