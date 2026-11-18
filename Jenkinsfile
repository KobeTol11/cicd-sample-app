// Pipeline voor de sample app (Lab 2: zelfstandig, zonder Freestyle-jobs)
node {
    stage('Checkout') {
        checkout scm
    }

    // Bouwt het image en start de container op de buildserver (staging).
    // sample-app.sh ruimt zelf eerst tempdir en de oude container op.
    stage('Build & deploy (staging)') {
        sh 'bash ./sample-app.sh'
    }

    // Zelfde test als de vroegere job TestSampleApp
    stage('Test (staging)') {
        sh '''
            APP_IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' samplerunning)
            JENKINS_IP=$(hostname -i)
            echo "App IP: ${APP_IP}, Jenkins IP: ${JENKINS_IP}"

            for i in $(seq 1 15); do
              if curl -s "http://${APP_IP}:5050/" | grep "You are calling me from ${JENKINS_IP}"; then
                echo "Test geslaagd"
                exit 0
              fi
              sleep 2
            done
            echo "Test gefaald, output was:"
            curl -s "http://${APP_IP}:5050/"
            exit 1
        '''
    }
}
