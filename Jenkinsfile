pipeline {
    agent any

    stages {
        stage('Validation') {
            steps {
                sh 'sudo nginx -t'
            }
        }
        stage('Reload') {
            steps {
                sh 'sudo nginx -s reload'
            }
        }
    }

    post {
        success {
            echo 'Configuration nginx appliquée avec succès.'
        }
        failure {
            echo 'Erreur dans la configuration nginx — reload annulé.'
        }
    }
}
