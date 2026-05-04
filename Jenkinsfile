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
                    echo "${params.DOCKER_IMAGE_VERSION}"

                }

            }
        }
    }
}
