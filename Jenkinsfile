properties([
    parameters([
        choice(name: 'TARGET', choices: ['both', 'staging', 'production'],
       description: 'Welk deel van de pipeline draaien?')
    ]),
    pipelineTriggers([githubPush()])
])

def target = params.TARGET ?: 'both'

node {
    if (target in ['staging', 'both']) {
        stage('Checkout') {
            checkout scm
        }
        stage('Preparation') {
            catchError(buildResult: 'SUCCESS', stageResult: 'SUCCESS') {
                sh 'docker stop samplerunning'
                sh 'docker rm samplerunning'
            }
        }
        stage('Build') {
            sh 'bash ./sample-app.sh'
        }
        stage('Test') {
            sh '''
                sleep 3
                APP_IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' samplerunning)
                JENKINS_IP=$(hostname -i | awk '{print $1}')
                OUT=$(curl -s http://$APP_IP:5050/)
                echo "$OUT"
                echo "$OUT" | grep "You are calling me from $JENKINS_IP"
            '''
        }
    }
     
    if (target in ['production', 'both']) {
        stage('Production: deploy') {
            withCredentials([usernamePassword(credentialsId: 'dmz010-ssh',
                                              usernameVariable: 'SSH_USER',
                                              passwordVariable: 'SSHPASS')]) {
                sh '''
                    sshpass -e ssh -o StrictHostKeyChecking=no $SSH_USER@$PROD_HOST \
                      "docker rm -f samplerunning 2>/dev/null || true; \
                       rm -rf app; \
                       git clone --branch main $SAMPLE_REPO app; \
                       cd app && bash ./sample-app.sh"
                '''
            }
        }
        stage('Production: verify') {
            sh 'sleep 3; curl -sf http://$PROD_HOST:5050/ | grep "You are calling me from"'
        }
    }
}