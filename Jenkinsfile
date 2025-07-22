pipeline {
    agent any

    stages {
        stage('Clone') {
            steps {
                git branch: 'main', url: 'https://github.com/anishSub/genkins'
            }
        }
        stage('Build') {
            steps {
                echo 'Build step goes here'
            }
        }
        stage('Test') {
            steps {
                echo 'Test step goes here'
            }
        }
    }
}
