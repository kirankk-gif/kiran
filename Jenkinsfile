pipeline {
    agent any

    environment {
        SERVER_IP   = '192.168.0.84'
        SSH_PORT    = '2222'
        DEPLOY_DIR  = '/var/www/kiran'
        NGINX_PORT  = '8083'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Check Files') {
            steps {
                sh '''
                    set -e

                    echo "Checking website files..."

                    test -f html/index.html

                    echo "index.html found successfully."
                '''
            }
        }

        stage('Deploy Website') {
            steps {
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'ssh-kiran-key',
                        keyFileVariable: 'SSH_KEY',
                        usernameVariable: 'SSH_USER'
                    )
                ]) {
                    sh '''
                        set -e

                        echo "Deploying index.html..."

                        scp -i "$SSH_KEY" \
                            -P "$SSH_PORT" \
                            html/index.html \
                            "${SSH_USER}@${SERVER_IP}:/tmp/index.html"

                        ssh -i "$SSH_KEY" \
                            -p "$SSH_PORT" \
                            "${SSH_USER}@${SERVER_IP}" \
                            "sudo -n install -d -m 755 ${DEPLOY_DIR} &&
                             sudo -n install -m 644 /tmp/index.html ${DEPLOY_DIR}/index.html"

                        echo "Website file deployed successfully."
                    '''
                }
            }
        }

        stage('Nginx Configuration') {
            steps {
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'ssh-kiran-key',
                        keyFileVariable: 'SSH_KEY',
                        usernameVariable: 'SSH_USER'
                    )
                ]) {
                    sh '''
                        set -e

                        echo "Configuring Nginx..."

                        ssh -i "$SSH_KEY" \
                            -p "$SSH_PORT" \
                            "${SSH_USER}@${SERVER_IP}" \
                            "sudo -n tee /etc/nginx/sites-available/kiran > /dev/null" <<NGINX
server {
    listen ${NGINX_PORT};
    server_name _;

    root ${DEPLOY_DIR};
    index index.html;

    location / {
        try_files \\$uri \\$uri/ =404;
    }
}
NGINX

                        echo "Enabling Nginx site..."

                        ssh -i "$SSH_KEY" \
                            -p "$SSH_PORT" \
                            "${SSH_USER}@${SERVER_IP}" \
                            "set -e
                             if [ ! -L /etc/nginx/sites-enabled/kiran ]; then
                                 sudo -n ln -s /etc/nginx/sites-available/kiran /etc/nginx/sites-enabled/kiran
                             fi
                             sudo -n nginx -t
                             if systemctl is-active --quiet nginx; then
                                 sudo -n systemctl reload nginx
                             else
                                 sudo -n systemctl start nginx
                             fi"

                        echo "Nginx configuration completed."
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    set -e

                    echo "Verifying website..."

                    curl --fail --show-error \
                        --connect-timeout 5 \
                        "http://${SERVER_IP}:${NGINX_PORT}/"

                    echo "Website verification completed."
                '''
            }
        }
    }

    post {
        success {
            echo 'HTML deployment completed successfully.'
        }

        failure {
            echo 'Deployment failed. Check Console Output for the failed stage.'
        }
    }
}
