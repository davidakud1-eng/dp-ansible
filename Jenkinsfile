pipeline {
    agent { label 'platform-ops' }

    options {
        disableConcurrentBuilds()
        timestamps()
    }

    environment {
        // ================================================================
        // Ansible Configuration
        // ================================================================

        ANSIBLE_CONFIG = "${WORKSPACE}/ansible.cfg"
        ANSIBLE_INVENTORY = "${WORKSPACE}/inventory/hosts-prod.yml"

        // ================================================================
        // EKS Runtime Configuration
        //
        // Overrides the safe Ansible role default:
        //
        // eks_runtime_enabled: false
        //
        // When true, EKS-dependent platform components such as the
        // AWS Load Balancer Controller and kube-prometheus-stack
        // are allowed to deploy.
        // ================================================================

        EKS_RUNTIME_ENABLED = 'true'
    }

    stages {

        // ==================================================================
        // Checkout
        // ==================================================================

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        // ==================================================================
        // Verify Ansible
        // ==================================================================

        stage('Verify Ansible') {
            steps {
                sh '''
                    ansible --version
                    ansible-playbook --version
                '''
            }
        }

        // ==================================================================
        // Syntax Check
        // ==================================================================

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
                          --extra-vars "eks_runtime_enabled=$EKS_RUNTIME_ENABLED" \
                          --syntax-check

                        rm -f .vault-pass
                    '''
                }
            }
        }

        // ==================================================================
        // Deploy DP Platform
        // ==================================================================

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
                          --vault-password-file .vault-pass \
                          --extra-vars "eks_runtime_enabled=$EKS_RUNTIME_ENABLED"

                        rm -f .vault-pass
                    '''
                }
            }
        }
    }

    post {

        // ==================================================================
        // Cleanup
        // ==================================================================

        always {
            sh '''
                rm -f .vault-pass
            '''
        }

        // ==================================================================
        // Pipeline Result
        // ==================================================================

        success {
            echo 'DP platform configuration completed successfully.'
        }

        failure {
            echo 'DP platform configuration failed.'
        }
    }
}