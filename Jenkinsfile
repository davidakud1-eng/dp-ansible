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
        // Must be enabled for the AWS Load Balancer Controller role
        // to execute.
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
        // ==================================================================

        stage('Runtime Configuration') {
            steps {
                sh '''
                    echo "=== DP ANSIBLE RUNTIME CONFIGURATION ==="
                    echo "EKS_RUNTIME_ENABLED=${EKS_RUNTIME_ENABLED}"
                    echo "ANSIBLE_INVENTORY=${ANSIBLE_INVENTORY}"
                    echo "PLAYBOOK=playbooks/platform_ops.yml"
                    echo "ANSIBLE_TAG=aws_load_balancer_controller"
                '''
            }
        }

        // ==================================================================
        // Syntax Check
        //
        // Validate the Platform Ops playbook containing the targeted role.
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
                          playbooks/platform_ops.yml \
                          --vault-password-file .vault-pass \
                          --extra-vars "eks_runtime_enabled=$EKS_RUNTIME_ENABLED" \
                          --syntax-check

                        rm -f .vault-pass
                    '''
                }
            }
        }

        // ==================================================================
        // Deploy AWS Load Balancer Controller
        //
        // Temporary targeted execution used to validate and repair the
        // existing AWS Load Balancer Controller Helm release.
        //
        // Only the role tagged:
        //
        //   aws_load_balancer_controller
        //
        // is executed.
        // ==================================================================

        stage('Deploy AWS Load Balancer Controller') {
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
                          playbooks/platform_ops.yml \
                          --vault-password-file .vault-pass \
                          --extra-vars "eks_runtime_enabled=$EKS_RUNTIME_ENABLED" \
                          --tags aws_load_balancer_controller

                        rm -f .vault-pass
                    '''
                }
            }
        }

        // ==================================================================
        // Verify AWS Load Balancer Controller
        // ==================================================================

        stage('Verify AWS Load Balancer Controller') {
            when {
                expression {
                    return params.EKS_RUNTIME_ENABLED
                }
            }

            steps {
                sh '''
                    echo "=== AWS LOAD BALANCER CONTROLLER DEPLOYMENT ==="

                    kubectl get deployment \
                      aws-load-balancer-controller \
                      --namespace kube-system

                    echo
                    echo "=== AWS LOAD BALANCER CONTROLLER PODS ==="

                    kubectl get pods \
                      --namespace kube-system \
                      --selector app.kubernetes.io/name=aws-load-balancer-controller \
                      -o wide
                '''
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
            echo 'DP AWS Load Balancer Controller deployment completed successfully.'
        }

        failure {
            echo 'DP AWS Load Balancer Controller deployment failed.'
        }
    }
}