pipeline {
    agent any //This needs to changed in case Jenkins runs in a VM or isolated from the target servers
    
    parameters{
        
        string(name: 'GIT_BRANCH',
               defaultValue: 'main',
               description: 'Git branch to be used for testing')

        
        choice( choices: ['4.6.16', '4.7.33', '4.8.14', '4.9.0', '4.10.10'], 
                description: 'OCP version to be installed',
                name: 'OCP_VERSION')

        choice( choices: ['skidoo', 'nova'], 
                description: 'Machine to deploy OCP',
                name: 'MACHINE')

        string(name: 'AI_VERSION',
               defaultValue: 'v1.0.24.2',
               description: 'Assisted Installer version to be used')

        booleanParam(name: 'OVW_CONFIG', 
                defaultValue: false, 
                description: 'If true, clustername and base DNS domain params will be overwritten')
                
        string(name: 'CLUSTER_NAME',
               defaultValue: 'clustername',
               description: 'Name of the cluster')
               
        string(name: 'BASE_DNS_DOMAIN',
               defaultValue: 'example.lab',
               description: 'Base DNS domain')

        booleanParam(name: 'KVM', 
                defaultValue: false, 
                description: 'If true, the deployment will be fully virtualized using KVM')

        string(name: 'VM_IMAGES_PATH',
               defaultValue: '/var/lib/jenkins/workspace/Crucible/libvirt/images',
               description: 'Only applies for KVM deployments.Path to place qcow2 images')

        /*booleanParam(name: 'VAULT_OW', 
                defaultValue: false, 
                description: 'If true, the deployment will be fully virtualized using KVM')
        */
        booleanParam(name: 'SSH_INV', 
                defaultValue: true, 
                description: 'If true, the inventory will be collected from a remote machine')

        string(name: 'SSH_INV_ADDR',
               defaultValue: 'oc-client.infra.cars.lab',
               description: 'Only applies if SSH_INV is true. Name or IP address for remote inventory collection.')

        string(name: 'INV_PATH',
               defaultValue: '/home/crucible/micosta/clean_inventories',
               description: 'Path for inventory file for remote or local machine')

        booleanParam(name: 'QA_TESTS', 
                defaultValue: false, 
                description: 'If true, it will run the QA tests')
    }
    
    stages {
        stage("Clean workdir, podman, VMs and RAM"){
            when{
                expression { this.params.KVM == true }
            }
            steps{
                sh "sudo podman pod rm -af" //Kill all pods for a clean start
                sh "sudo sync"
                dir("${VM_IMAGES_PATH}"){ //This can trigger issues if the path changes
                    script{//Stop existing VMs
                        try {
                            def files = findFiles(glob: '*.qcow2')
                            for(File file: files){
                                def vm_name = file.name - '_main.qcow2'
                                echo vm_name
                                sh 'sudo virsh destroy '+vm_name
                            }
                        }catch (e){
                            println 'Not qcow2 images found'
                        }
                    }
                }
 
                sh "sudo rm -rf ./*"
                sh "sudo rm -rf /tmp/*" //Wipe /tmp to avoid storage issues
                sh "sudo sh -c \"/usr/bin/echo 3 > /proc/sys/vm/drop_caches\"" //Wipe the RAM
            }
        }
        stage("Clone git repo and prepare configuration"){
            steps{
                git branch: GIT_BRANCH, url: 'https://github.com/nocturnalastro/crucible'
            }
        }
        stage("Collect Inventory and Vault from remote folder"){
            when{
                expression { this.params.SSH_INV == true }
            }
            steps{
                script{
                    sshagent(['bastion_ssh']) {
                        sh '''
                            scp -o StrictHostKeyChecking=no \"crucible@${SSH_INV_ADDR}:${INV_PATH}/${MACHINE}.yml\" ./inventory.yml
                            scp -o StrictHostKeyChecking=no \"crucible@${SSH_INV_ADDR}:${INV_PATH}/*vault*.yml\" ./inventory.vault.yml
                        '''
                    }
                }

            }
        }
        stage("Prepare inventory"){
            steps{
                /*script{
                    echo 'Copying configuration file from repo'
                    def files = findFiles(glob: 'jenkins/*.yml')
                    for(File file: files){
                        sh 'cp '+file.path+' ./'+file.name
                    }
                }*/
                //Classic unix style...
                sh 'sed -i \"s!openshift_full_version:.*!openshift_full_version: ${OCP_VERSION}!g\" inventory.yml'
                sh 'sed -i \"s!ai_version:.*!ai_version: ${AI_VERSION}!g\" ./roles/setup_assisted_installer/defaults/main.yml'
                sh 'sed -i \"s!ansible_host: 10.40.0.245!ansible_connection: local!g\" inventory.yml'
                sh 'sed -i \"s!cluster_name:.*!cluster_name: ${MACHINE}!g\" inventory.yml'
                sh 'sed -i \"s!setup_registry_service:.*!&\\n    setup_assisted_installer: False!\" inventory.yml'
                 
                script{ //Patching paths...
                    sh 'sed -i \"s!repo_root_path:.*!repo_root_path: '+pwd()+'!g\" inventory.yml'
                    sh 'sed -i \"s!ssh_key_dest_base_dir:.*!ssh_key_dest_base_dir: '+pwd()+'!g\" inventory.yml'
                    sh 'sed -i \"s!kubeconfig_dest_dir:.*!kubeconfig_dest_dir: '+pwd()+'!g\" inventory.yml'
                    if (this.params.OVW_CONFIG) {
                        sh 'sed -i \"s!cluster_name:.*!cluster_name: ${CLUSTER_NAME}!g\" inventory.yml'
                        sh 'sed -i \"s!base_dns_domain:.*!base_dns_domain: ${BASE_DNS_DOMAIN}!g\" inventory.yml' 
                    }
                    if(this.params.KVM){
                        sh 'sed -i \"s!vm_bridge_interface:.*!vm_bridge_interface: ens1f1!g\" inventory.yml'
                        sh 'sed -i \"s!images_dir: .*!images_dir: ${VM_IMAGES_PATH}!g\" inventory.yml'
                    }
                }
            }
        }

/*        stage("Installing Crucible dependencies"){ //This step is optional, we can assume these dependencies are present for jenkins user
            steps{
                dir('crucible') {
                    sh "ansible-galaxy collection install -r requirements.yml"
                }
            }
        }*/
        stage("Adding pull-secret file and ssh keys"){
            steps{
                script{
                    try{
                        sh 'mkdir fetched'
                        writeFile file: 'fetched/pull-secret.txt', text: //Insert here your pull-secret
                        writeFile file: 'ssh_public_key.pub', text: //Insert here your ssh_public_key
                    }
                    catch (e) {
                        println "Files already exist. Skipping..."
                    }
                }

            }
        }
        stage("Validate Prerequirements and Inventory"){
            steps{
                ansiColor('xterm') {
                    ansiblePlaybook(
                        vaultCredentialsId: 'crucible_vault',
                        playbook: 'prereq_facts_check.yml', 
                        inventory: 'inventory.yml', 
                        extras: "-e \"@inventory.vault.yml\"",
                        colorized: true
                    )
                    ansiblePlaybook(
                        vaultCredentialsId: 'crucible_vault',
                        playbook: 'playbooks/validate_inventory.yml', 
                        inventory: 'inventory.yml', 
                        extras: "-e \"@inventory.vault.yml\"",
                        colorized: true
                    )
                }
            }
        }

        stage('Deploying deploy_prerequisites.yml') { 
            steps {
                ansiColor('xterm') {
                    ansiblePlaybook(
                        vaultCredentialsId: 'crucible_vault',
                        playbook: 'deploy_prerequisites.yml', 
                        inventory: 'inventory.yml', 
                        extras: "-e \"@inventory.vault.yml\"",
                        colorized: true
                    )
                 }
            }
        }
        stage('Deploying deploy_cluster.yml') { 
            steps {
                ansiColor('xterm') {
                    ansiblePlaybook(
                        vaultCredentialsId: 'crucible_vault',
                        playbook: 'deploy_cluster.yml', 
                        inventory: 'inventory.yml', 
                        extras: "-e \"@inventory.vault.yml\" -vv",
                        colorized: true
                    )
                }
            }
        }
        stage('Deploying post_install.yml') { 
            steps {
                script{
                    try {
                        // Fails with non-zero exit if dir1 does not exist
                        def cluster_id = sh(script:'cat fetched/cluster.txt', returnStdout:true).trim()
                        ansiColor('xterm') {
                            ansiblePlaybook(
                                vaultCredentialsId: 'crucible_vault',
                                playbook: 'post_install.yml', 
                                inventory: 'inventory.yml', 
                                extras: "-e \"@inventory.vault.yml\" -e cluster_id=${cluster_id}",
                                colorized: true
                            )
                        }
                    } catch (Exception ex) {
                        println("Unable to read cluster.txt: ${ex}")
                    }
                }
                
            }
        }
        stage('Testing environment') { 
            steps {
                ansiColor('xterm') {
                    ansiblePlaybook(
                        vaultCredentialsId: 'crucible_vault',
                        playbook: 'tests/run_tests.yml', 
                        inventory: 'inventory.yml', 
                        extras: "-e \"@inventory.vault.yml\"",
                        colorized: true 
                    )
                }
            }
        }
        stage('QA Sanity Tests'){
            when{
                expression { this.params.QA_TESTS == true }
            }

            steps {
                script {

                    TESTS_EXECUTED = sh(returnStdout: true, script: "echo EXECUTED").trim()

                    try {
                        timeout(time: 2, unit: 'HOURS') {
                            sh '''
                                . $OCP_EDGE_VENV/bin/activate

                                export ASSISTED_INSTALLER="${ASSISTED_INSTALLER}";
                                export SNO="${SNO}";
                                export PROVISIONING_NET_IPV6="${PROVISIONING_NET_IPV6}";
                                export OCP_EDGE_AUTO_REPO="${OCP_EDGE_AUTO_REPO}";
                                export OCP_EDGE_AUTO_BRANCH="${OCP_EDGE_AUTO_BRANCH}";
                                export RSA_ID_PATH="${HOME}/.ssh/id_rsa";
                                export RP_PROJECT="${RP_PROJECT}";
                                export SSH_USER_NAME="${USER}";
                                export HOME_PATH="${HOME}";
                                export SKIP_TEARDOWN="${SKIP_TEARDOWN}";
                                export DEBUG_MODE=${DEBUG_MODE};

                                if [ $BM_ENV == 'true' ]; then
                                    if [ "${VSPHERE}" == "true" ]; then
                                        export PROVISIONHOST="${HOST}";
                                    else
                                        export PROVISIONHOST="${HOSTNAME}";
                                    fi
                                    export ENVIRONMENT="Baremetal";
                                else
                                    export ENVIRONMENT="Virtual";
                                    if [ "${ASSISTED_INSTALLER}" == "false" ] && [ "${SNO}" == "false" ]; then
                                        export PROVISIONHOST="provisionhost-0-0";
                                    else
                                        if [ "${VSPHERE}" == "true" ]; then
                                            export PROVISIONHOST="${HOST}";
                                        else
                                            export PROVISIONHOST="${HOSTNAME}";
                                        fi
                                        if [ ${USER} != 'root' ]; then
                                            sudo cp /root/kubeadmin-password $HOME/
                                            sudo cp /root/install-config.yaml $HOME/
                                            sudo cp /root/${KUBECONFIG_NAME} $HOME/
                                            if [ -f /root/combined-secret.json ]; then
                                                sudo cp /root/combined-secret.json $HOME/
                                            fi
                                        fi
                                    fi
                                fi

                                if [ "${ASSISTED_INSTALLER}" == "true" ] && [ "${SNO}" == "false" ]; then
                                    cp ${HOME}/${KUBECONFIG_NAME} $BASE_REPO_DIR/
                                    export SCENARIO_NAME=scenario_assisted_sanity.yaml;
                                elif [ "${SNO}" == "true" ]; then
                                    cp ${HOME}/${KUBECONFIG_NAME} $BASE_REPO_DIR/
                                    export SCENARIO_NAME=scenario_sno_sanity.yaml;
                                else
                                    cp ${HOME}/clusterconfigs/auth/${KUBECONFIG_NAME} $BASE_REPO_DIR/
                                    export SCENARIO_NAME=scenario_sanity.yaml;
                                fi

                                export KUBECONFIG=$BASE_REPO_DIR/${KUBECONFIG_NAME}
                                export CLUSTER_NAME=$(oc config view -o jsonpath='{.clusters[-1].name}')
                                export OCP_EDGE_AUTO_PATH="${HOME}/ocp-edge-auto_${CLUSTER_NAME}";
                                export OCP_EDGE_AUTO_RESULTS_PATH="/var/tmp/ocp-edge-auto-results_${CLUSTER_NAME}";
                                export ORIG_KUBECONFIG_NAME=${KUBECONFIG_NAME};

                                pushd $TEFLO_SCENARIOS_DIR;
                                teflo run --data-folder data -s ${SCENARIO_NAME} -t provision;
                                teflo run --data-folder data -s ${SCENARIO_NAME} -t orchestrate;
                                teflo run --data-folder data -s ${SCENARIO_NAME} -t execute;
                                if [ "${TEFLO_TEST_REPORTING}" == "true" ]; then
                                    teflo run --data-folder data -s ${SCENARIO_NAME} -t report;
                                fi

                                popd
                            '''
                        } // end timeout
                    }
                    catch (deployment_err) {
                        println "Error while executing 'Sanity Test' ${deployment_err}"
                        currentBuild.result = "UNSTABLE"
                    }
                }// end script

            } // end steps  
        }
    }
    post{
        always{
            script{
                // Default values
                def subject = "${currentBuild.currentResult} Job: '${env.JOB_NAME} [${env.BUILD_NUMBER} for OCP ${OCP_VERSION}]'"
                def summary = "${subject} (${env.BUILD_URL})"
                def details = """RESULTS: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]':\n
                    Check console output at '${env.BUILD_URL}', ${env.JOB_NAME} [${env.BUILD_NUMBER}]\n"""
                
                emailext (
                    subject: subject,
                    body: details,
                    attachLog: true,
                    compressLog: true,
                   // to: "cdonato@redhat.com, rpollak@redhat.com", //To be changed :)
                )   
            }
        }
    }
}
