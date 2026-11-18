// Pipeline voor de sample app
//   staging   = Docker op de buildserver (infra010)
//   productie = Docker op de webserver (dmz010), achter nginx op sample.infra.lan
// PROD_HOST komt uit de Jenkins-configuratie (JCasC, beheerd door Ansible).
node {
    stage('Checkout') {
        checkout scm
    }

    // Bouwt het image en start de container op de buildserver (staging).
    // sample-app.sh ruimt zelf eerst tempdir en de oude container op.
    stage('Build & deploy (staging)') {
        sh 'bash ./sample-app.sh'
    }

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

    // Docker op afstand via SSH: DOCKER_HOST=ssh://... laat de docker-CLI
    // verbinden met de Docker-daemon op dmz010, als gebruiker 'deploy'.
    // De container luistert enkel op 127.0.0.1: alleen nginx kan erbij.
    stage('Deploy (production)') {
        sshagent(credentials: ['deploy-ssh']) {
            sh '''
                mkdir -p ~/.ssh
                ssh-keygen -F "${PROD_HOST}" > /dev/null || ssh-keyscan -H "${PROD_HOST}" >> ~/.ssh/known_hosts
                export DOCKER_HOST="ssh://deploy@${PROD_HOST}"

                docker build -t sampleapp tempdir
                docker rm -f samplerunning || true
                docker run -d --name samplerunning --restart unless-stopped \
                  -p 127.0.0.1:5050:5050 sampleapp
            '''
        }
    }

    // Test via nginx en HTTPS, zoals een gebruiker de app bereikt.
    // --resolve: er is (nog) geen DNS, dus de naam koppelen aan het IP.
    // -k: het certificaat is zelfondertekend.
    stage('Test (production)') {
        sh '''
            for i in $(seq 1 15); do
              if curl -sk --resolve "sample.infra.lan:443:${PROD_HOST}" https://sample.infra.lan/ | grep "You are calling me from"; then
                echo "Test geslaagd: app bereikbaar via https://sample.infra.lan"
                exit 0
              fi
              sleep 2
            done
            echo "Test gefaald"
            exit 1
        '''
    }
}
