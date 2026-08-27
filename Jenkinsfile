pipeline {
    agent { label 'platform-ops' }

    options {
        disableConcurrentBuilds()
        timestamps()
    }

    parameters {

        // ================================================================
        // EKS Runtime Control
        //
        // Disabled by default so normal Ansible runs do not attempt to
        // configure EKS-dependent platform components when the ephemeral
        // EKS runtime does not exist.
        //
        // When enabled, EKS-dependent components such as the AWS Load
        // Balancer Controller and kube-prometheus-stack are allowed
        // to deploy.
        // ================================================================

        booleanParam(
            name: 'EKS_RUNTIME_ENABLED',
            defaultValue: false,
            description: 'Deploy EKS-dependent DP platform components.'
        )
    }

    environment {

        // ================================================================
        // Ansible Configuration
        // ================================================================

        ANSIBLE_CONFIG = "${WORKSPACE}/ansible.cfg"
        ANSIBLE_INVENTORY = "${WORKSPACE}/inventory/hosts-prod.yml"
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
                    echo "=== ANSIBLE ==="
                    ansible --version

                    echo
                    echo "=== ANSIBLE PLAYBOOK ==="
                    ansible-playbook --version
                '''
            }
        }

        // ==================================================================
        // Display Runtime Configuration
        //
        // Makes the selected EKS runtime state visible in the Jenkins
        // console before validation or deployment begins.
        // ==================================================================

        stage('Runtime Configuration') {
            steps {
                sh '''
                    echo "=== DP ANSIBLE RUNTIME CONFIGURATION ==="
                    echo "EKS_RUNTIME_ENABLED=${EKS_RUNTIME_ENABLED}"
                    echo "ANSIBLE_INVENTORY=${ANSIBLE_INVENTORY}"
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
        //
        // EKS-dependent roles are controlled by eks_runtime_enabled.
        //
        // When false:
        //   - normal EC2/platform configuration can run
        //   - EKS-dependent roles remain disabled
        //
        // When true:
        //   - AWS Load Balancer Controller can deploy
        //   - kube-prometheus-stack can deploy
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
        //
        // Ensures the temporary Vault password file is removed even if
        // Ansible fails before its normal cleanup command is reached.
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