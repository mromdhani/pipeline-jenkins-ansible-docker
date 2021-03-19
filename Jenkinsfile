pipeline {
    agent any
    
    tools  {
        maven "my-maven"
    }
    environment {
        NEXUS_VERSION = "nexus3"
        NEXUS_PROTOCOL = "http"
        NEXUS_URL = "localhost:8081"
        NEXUS_REPOSITORY_SNAPSHOT = "mylocalrepo-snapshots"
        NEXUS_REPOSITORY_RELEASE = "mylocalrepo-releases"
        NEXUS_CREDENTIAL_ID = "nexus-user-credentials"    
    }

    stages {

        // Clone from Git
        stage("Clone App from Git"){
             steps{
                
                echo "====++++  Clone App from Git ++++===="
                git branch:"master", url: "https://github.com/mromdhani/pipeline-jenkins-ansible-docker.git"
            } 

         
        }
        // Build and Unit Test (Maven/JUnit)
        stage("Build and Package"){
            steps{
                echo "====++++  Build and Unit Test (Maven/JUnit) ++++===="
                sh "mvn -f greetings-app/pom.xml  clean package"
            }           
        }  
        // Static Code Analysis (SonarQube)
        stage("Static Code Analysis (SonarQube)"){
        
            steps{
                withSonarQubeEnv('my-sonarqube-in-docker'){
                       echo "====++++  Static Code Analysis (SonarQube) ++++===="               
                        sh "mvn -f greetings-app/pom.xml clean package -Dsurefire.skip=true sonar:sonar -Dsonar.host.url=http://localhost:9000   -Dsonar.projectName=pipeline-jenkins-vagrant-ansible-app -Dsonar.projectVersion=$BUILD_NUMBER";
                }
            }  
        }         
        //172.17.0.1

        stage("Checking the Quality Gate") {
            steps {
                echo "====++++  Checking the returned SonarQube Quality Gate ++++===="
                timeout(time: 1, unit: 'HOURS') {
                    // Parameter indicates whether to set pipeline to UNSTABLE if Quality Gate fails
                    // true = set pipeline to UNSTABLE, false = don't
                    waitForQualityGate abortPipeline: true
                }
            }
        }   
        // Publication de l'artifact (Snaphot) sur Nexus3
        stage("Publish to Nexus Repository Manager (SNAPSHOT)") {
            steps {
                echo "====++++  Publish to Nexus Repository Manager ++++===="
                script {
                    pom = readMavenPom file: "greetings-app/pom.xml";
                    filesByGlob = findFiles(glob: "greetings-app/target/*.${pom.packaging}");
                    echo "${filesByGlob[0].name} ${filesByGlob[0].path} ${filesByGlob[0].directory} ${filesByGlob[0].length} ${filesByGlob[0].lastModified}"
                    artifactPath = filesByGlob[0].path;
                    artifactExists = fileExists artifactPath;
                    if(artifactExists) {
                        echo "*** File: ${artifactPath}, group: ${pom.groupId}, packaging: ${pom.packaging}, version ${pom.version}";
                        nexusArtifactUploader(
                            nexusVersion: NEXUS_VERSION,
                            protocol: NEXUS_PROTOCOL,
                            nexusUrl: NEXUS_URL,
                            groupId: pom.groupId,
                            version: pom.version,
                            repository: NEXUS_REPOSITORY_SNAPSHOT,
                            credentialsId: NEXUS_CREDENTIAL_ID,
                            artifacts: [
                                [artifactId: pom.artifactId,
                                classifier: '',
                                file: artifactPath,
                                type: pom.packaging],
                                [artifactId: pom.artifactId,
                                classifier: '',
                                file: "greetings-app/pom.xml",
                                type: "pom"]
                            ]
                        );
                    } else {
                        error "*** File: ${artifactPath}, could not be found";
                    }
                }
            }
        }  
    
         // Deploiement du WAR sur le server-staging avec Ansible
        stage("Deploy WAR on staging using Ansible"){
            steps{
                
                echo "====++++  Deploy WAR on staging using Ansible ++++===="
      
                ansiblePlaybook(  credentialsId: 'ssh-key-for-server-staging', 
                                  inventory:  "$WORKSPACE/config-as-code/ansible/hosts-jenkins", 
                                  playbook: '$WORKSPACE/config-as-code/ansible/playbook-deploy-staging.yaml')          
            } 
        }
            
        // Do performance tests on the app
	    stage('Do the Performance Testing'){	   	   
	   	    steps {  
               echo "====++++ Do the performance tests Against the Staging Server ++++===="        
               sh "mvn -f greetings-app/pom.xml  clean verify -Dmaven.test.skip=true"   // This triggers JMeter Performance Analysis and generates results in target/jmeter/results          
            }
	        
        }
        // Publish the test reports
	    stage ('Publish the performance reports') {
	        steps {
               echo "====++++ Publish the performance Reports and Check the Threasholds ++++===="   
	           perfReport sourceDataFiles: "$WORKSPACE/greetings-app/target/jmeter/results/*.csv"	     
	        }
	    }
        // Promote the app On NEXUS from the SNAPSHOT -> RELEASE
        stage("Promote the app in  Nexus Repository Manager") {
            steps {
                echo "====++++  Promote the app in  Nexus Repository Manager ++++===="
                sh "mv greetings-app/target/greetings-0.1-SNAPSHOT.war greetings-app/target/greetings-0.1-RELEASE.war"
                script {
                    pom = readMavenPom file: "greetings-app/pom.xml";
                    filesByGlob = findFiles(glob: "greetings-app/target/*.${pom.packaging}");
                    echo "${filesByGlob[0].name} ${filesByGlob[0].path} ${filesByGlob[0].directory} ${filesByGlob[0].length} ${filesByGlob[0].lastModified}"
                    artifactPath = filesByGlob[0].path;                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           
                    artifactVersion ="0.1-RELEASE-${BUILD_NUMBER}";  //Append the build number for traceability
                    artifactExists = fileExists artifactPath;
                    if(artifactExists) {
                        echo "*** File: ${artifactPath}, group: ${pom.groupId}, packaging: ${pom.packaging}, version ${pom.version}";
                        nexusArtifactUploader(
                            nexusVersion: NEXUS_VERSION,
                            protocol: NEXUS_PROTOCOL,
                            nexusUrl: NEXUS_URL,
                            groupId: pom.groupId,
                            version: artifactVersion,
                            repository: NEXUS_REPOSITORY_RELEASE,
                            credentialsId: NEXUS_CREDENTIAL_ID,
                            artifacts: [
                                [artifactId: pom.artifactId,
                                classifier: '',
                                file: artifactPath,
                                type: pom.packaging],
                                [artifactId: pom.artifactId,
                                classifier: '',
                                file: "greetings-app/pom.xml",
                                type: "pom"]
                            ]
                        );
                    } else {
                        error "*** File: ${artifactPath}, could not be found";
                    }
                }
            }
        }
        // Deploiement du WAR sur le server-production avec Ansible
        stage("Deploy WAR on Production using Ansible"){
            steps{
                
                echo "====++++  Deploy WAR on ***Production*** using Ansible ++++===="
      
                ansiblePlaybook(  credentialsId: 'ssh-key-for-server-production', 
                                  inventory:  "$WORKSPACE/config-as-code/ansible/hosts-jenkins", 
                                  playbook: '$WORKSPACE/config-as-code/ansible/playbook-deploy-production.yaml')          
            } 
        }
       
    }
}