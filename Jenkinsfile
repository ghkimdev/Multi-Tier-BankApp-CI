pipeline {
    agent any
    
    parameters {
        string(name: "DOCKER_TAG", defaultValue: 'latest', description: 'Docker tags')
    }
    
    tools {
        maven 'maven3'
    }
    
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    
    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/ghkimdev/Multi-Tier-BankApp-CI.git'
            }
        }
        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }
        stage('test') {
            steps {
                sh "mvn test -DskipTests=true"
            }
        }
        /*
        stage('File System Scan') {
            steps {
                sh "trivy fs --format table -o fs.html ."
            }
        }
        */
        stage('Sonar Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=bankapp -Dsonar.projectKey=bankapp \
                    -Dsonar.java.binaries=target '''
                }
            }
        }
        stage('Building & Publish Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'maven-setting-devopsshack', jdk: '', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                    sh "mvn deploy -DskipTests=true"
                }
            }
        }
        stage('Docker Build & Tag') {
            steps {
                withDockerRegistry(credentialsId: 'docker-cred') {
                    sh "docker build -t ghkimdev/bankapp:${params.DOCKER_TAG} ."
                }
            }
        }
        /*
        stage('Docker Image Scan') {
            steps {
                sh "trivy image --format table -o image.html ghkimdev/bankapp:${params.DOCKER_TAG}"
            }
        }
        */
        stage('Docker Push') {
            steps {
                withDockerRegistry(credentialsId: 'docker-cred') {
                    sh "docker push ghkimdev/bankapp:${params.DOCKER_TAG}"
                }
            }
        }
        stage('Update YAML Manifest in Other Repo') {
            steps {
                script {
                    withCredentials([gitUsernamePassword(credentialsId: 'git-cred', gitToolName: 'Default')]) {
                        sh '''
                        #Clone The repo
                        rm -rf Multi-Tier-BankApp-CD
                        git clone https://github.com/ghkimdev/Multi-Tier-BankApp-CD.git
                        cd Multi-Tier-BankApp-CD
                        
                        #List files to confirm the presence of bankapp-ds.yml
                        ls -l bankapp
                        
                        #Get the absolute path for the current directory
                        repo_dir=$(pwd)
                        
                        #Use the absolute path for sed
                        sed -i 's|image: ghkimdev/bankapp:.*|image: ghkimdev/bankapp:'${DOCKER_TAG}'|' ${repo_dir}/bankapp/bankapp-ds.yml
                        '''
                        
                        //Confirm the change
                        sh '''
                        echo "Update YAML file contents:"
                        cat Multi-Tier-BankApp-CD/bankapp/bankapp-ds.yml
                        '''
                        // Configure Git for committing changes and pushing
                        sh '''
                        git config user.email "ghkim.dev@gmail.com"
                        git config user.name "ghkimdev"
                        '''
                        
                        // commit and push the updated YAML file back to the other repository
                        sh '''
                        cd Multi-Tier-BankApp-CD
                        ls
                        git add bankapp/bankapp-ds.yml
                        git commit -m "Update image tag to ${DOCKER_TAG}"
                        git push origin main
                        '''
                    }
                }
            }
        }
    }
}
