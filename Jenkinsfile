pipeline {
    agent any


    parameters {

        string(name: 'DOCKER_IMAGE_VERSION', defaultValue: '', description: 'Docker Image Version')

    }

    
    stages {
        stage('update deploy.yaml') {
            steps {
                // Jenkins 파이프라인에서 작업 디렉터리를 변경할 때 사용한다.
                dir('department-api') {
                    sh 'pwd'
                    sh 'ls -al'
                    echo "Received Docker Image Version : ${params.DOCKER_IMAGE_VERSION}"
                    sh 'git checkout main'
                    sh "sed -i 's|jin604/department-service:.*|jin604/department-service:${params.DOCKER_IMAGE_VERSION}|g' deploy.yaml"
                    sh 'cat deploy.yaml'

                }

            }
        }

        stage('Commit & Push') {
            steps {

                sh 'git status'
                sh 'git config --list'
                sh 'git config user.name "jin605"'
                sh 'git config user.email "jinddd3@gmail.com"'
                sh 'git config --list'
                sh 'git add .'
                sh "git commit -m 'Update Image Version ${params.DOCKER_IMAGE_VERSION}'"
                sh 'git status'

                sshagent(['github-university-app']) {

                    sh 'git push'

                }

            }
        }
    }
}
