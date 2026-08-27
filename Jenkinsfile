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
        // Must be enabled for the kube-prometheus-stack role
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
        // Install Ansible Collections
        //
        // Installs the Ansible collection dependencies declared in
        // requirements.yml into the Jenkins workspace on the Platform Ops
        // agent.
        //
        // This makes the pipeline reproducible and prevents deployments
        // from depending on manually installed Ansible collections.
        // ==================================================================

        stage('Install Ansible Collections') {
            steps {
                sh '''
                    echo "=== INSTALL ANSIBLE COLLECTIONS ==="

                    ansible-galaxy collection install \
                      -r requirements.yml \
                      -p ./collections

                    echo
                    echo "=== INSTALLED ANSIBLE COLLECTIONS ==="

                    ansible-galaxy collection list
                '''
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
                    echo "ANSIBLE_TAG=kube_prometheus_stack"
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
        // Deploy kube-prometheus-stack
        //
        // Temporary targeted execution used to validate the
        // kube-prometheus-stack Helm release.
        //
        // Only the role tagged:
        //
        //   kube_prometheus_stack
        //
        // is executed.
        // ==================================================================

        stage('Deploy kube-prometheus-stack') {
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
                          --tags kube_prometheus_stack

                        rm -f .vault-pass
                    '''
                }
            }
        }

        // ==================================================================
        // Verify kube-prometheus-stack
        //
        // Verify the Helm release and the major monitoring resources:
        //   - Prometheus
        //   - Grafana
        //   - Alertmanager
        //   - Prometheus Operator
        //   - kube-state-metrics
        //   - node-exporter
        //   - Prometheus persistent storage
        // ==================================================================

        stage('Verify kube-prometheus-stack') {
            when {
                expression {
                    return params.EKS_RUNTIME_ENABLED
                }
            }

            steps {
                sh '''
                    echo "=== MONITORING NAMESPACE ==="
                    kubectl get namespace monitoring

                    echo
                    echo "=== KUBE-PROMETHEUS-STACK HELM RELEASE ==="
                    helm list \
                      --namespace monitoring

                    echo
                    echo "=== MONITORING PODS ==="
                    kubectl get pods \
                      --namespace monitoring \
                      -o wide

                    echo
                    echo "=== MONITORING SERVICES ==="
                    kubectl get services \
                      --namespace monitoring

                    echo
                    echo "=== PROMETHEUS PVC ==="
                    kubectl get pvc \
                      --namespace monitoring
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
            echo 'DP kube-prometheus-stack deployment completed successfully.'
        }

        failure {
            echo 'DP kube-prometheus-stack deployment failed.'
        }
    }
}