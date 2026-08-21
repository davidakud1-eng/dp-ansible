pipeline {
    agent { label 'platform-ops' }

    options {
        disableConcurrentBuilds()
        timestamps()
    }

    environment {
        ANSIBLE_CONFIG = "${WORKSPACE}/ansible.cfg"
        ANSIBLE_INVENTORY = "${WORKSPACE}/inventory/hosts-prod.yml"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Verify Ansible') {
            steps {
                sh '''
                    ansible --version
                    ansible-playbook --version
                '''
            }
        }

        stage('Syntax Check') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'ansible-vault-password',
                        variable: 'ANSIBLE_VAULT_PASSWORD'
                    )
                ]) {
                    sh '''
                        set +x
                        umask 077

                        printf '%s' "$ANSIBLE_VAULT_PASSWORD" > .vault-pass

                        ansible-playbook \
                          -i "$ANSIBLE_INVENTORY" \
                          playbooks/site.yml \
                          --vault-password-file .vault-pass \
                          --syntax-check

                        rm -f .vault-pass
                    '''
                }
            }
        }

        stage('Deploy DP Platform') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'ansible-vault-password',
                        variable: 'ANSIBLE_VAULT_PASSWORD'
                    )
                ]) {
                    sh '''
                        set +x
                        umask 077

                        printf '%s' "$ANSIBLE_VAULT_PASSWORD" > .vault-pass

                        ansible-playbook \
                          -i "$ANSIBLE_INVENTORY" \
                          playbooks/site.yml \
                          --vault-password-file .vault-pass

                        rm -f .vault-pass
                    '''
                }
            }
        }
    }

    post {
        always {
            sh '''
                rm -f .vault-pass
            '''
        }

        success {
            echo 'DP platform configuration completed successfully.'
        }

        failure {
            echo 'DP platform configuration failed.'
        }
    }
}