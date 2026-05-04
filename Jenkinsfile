pipeline {
    agent any


    parameters: [

        string(name: 'DOCKER_IMAGE_VERSION', defaultvalue: '', description: 'Docker Image Version')
    ]

    
    stages {
        stage('Hello') {
            steps {
                sh 'pwd'
                sh 'ls -al'
                echo "${params.DOCKER_IMAGE_VERSION}"
            }
        }
    }
}
