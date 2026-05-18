// ============================================================
// Jenkinsfile — MySQL InnoDB Cluster 3-Node Deployment
// ============================================================
pipeline {
    agent any

    parameters {
        // ── Node IPs ──────────────────────────────────────────
        string(name: 'NODE1_IP',        defaultValue: '',          description: 'Primary node IP address')
        string(name: 'NODE2_IP',        defaultValue: '',          description: 'Secondary node 1 IP address')
        string(name: 'NODE3_IP',        defaultValue: '',          description: 'Secondary node 2 IP address')

        // ── MySQL Version & S3 Binaries ───────────────────────
        choice(name: 'MYSQL_VERSION',   choices: ['8.0', '8.4'],   description: 'MySQL major version to install')
        choice(name: 'MYSQL_MINOR',
            choices: [
                '8.0.42-33',   // Percona-Server-8.0.42-33
                '8.0.40-31',
                '8.4.4-4'
            ],
            description: 'Exact Percona Server minor version (auto-selects S3 path)')

        // ── Port & Data Directory ─────────────────────────────
        string(name: 'MYSQL_PORT',      defaultValue: '3306',      description: 'MySQL listener port')
        string(name: 'MYSQL_GR_PORT',   defaultValue: '33106',     description: 'Group Replication communication port')
        string(name: 'MYSQL_DATA_DIR',  defaultValue: '/dbdata/mysql', description: 'Base data directory on all nodes')

        // ── Cluster Identity ──────────────────────────────────
        string(name: 'APP_NAME',        defaultValue: 'myapp',     description: 'Application name (used in paths & cluster name)')
        string(name: 'ENV_NAME',        defaultValue: 'poc',       description: 'Environment (poc / dev / qa / prod)')
        string(name: 'DC_NAME',         defaultValue: 'dc1',       description: 'Data-centre identifier')

        // ── Credentials (Jenkins Credential IDs) ─────────────
        string(name: 'SSH_CRED_ID',     defaultValue: 'mysql-ssh-key',    description: 'Jenkins SSH credential ID for target nodes')
        string(name: 'SUDO_PASS_CRED',  defaultValue: 'mysql-sudo-pass',  description: 'Jenkins secret-text credential ID for sudo password')

        // ── Pipeline control ──────────────────────────────────
        booleanParam(name: 'DRY_RUN',   defaultValue: true,        description: 'Run in --check mode only (no changes)')
        choice(name: 'ANSIBLE_TAGS',
            choices: [
                'all',
                'install_dependencies',
                'download_binaries',
                'install_mysql',
                'setup_group_replication',
                'configure_cluster'
            ],
            description: 'Run only a specific Ansible tag (select "all" for full deployment)')
    }

    environment {
        // Derived from parameters — available throughout stages
        WORK_DIR          = "ansible"
        INVENTORY_FILE    = "${WORK_DIR}/inventory/hosts.ini"
        VARS_FILE         = "${WORK_DIR}/inventory/vars.yml"
        PLAYBOOK_MAIN     = "${WORK_DIR}/playbooks/main.yml"

        // S3 bucket paths keyed by MYSQL_MINOR
        S3_BASE           = "s3://your-org-mysql-binaries"  // ← change to your bucket
    }

    stages {

        // ----------------------------------------------------------
        stage('Validate Parameters') {
        // ----------------------------------------------------------
            steps {
                script {
                    def errors = []
                    ['NODE1_IP','NODE2_IP','NODE3_IP'].each { p ->
                        def val = params[p]
                        if (!val || !val.matches(/\d+\.\d+\.\d+\.\d+/)) {
                            errors << "${p} must be a valid IPv4 address (got: '${val}')"
                        }
                    }
                    if (params.NODE1_IP == params.NODE2_IP || params.NODE1_IP == params.NODE3_IP || params.NODE2_IP == params.NODE3_IP) {
                        errors << "NODE1_IP, NODE2_IP, NODE3_IP must all be different"
                    }
                    if (errors) { error(errors.join('\n')) }
                    echo "✅ All parameters validated"
                }
            }
        }

        // ----------------------------------------------------------
        stage('Resolve S3 Paths') {
        // ----------------------------------------------------------
            steps {
                script {
                    // Map minor version → S3 object keys
                    def s3Map = [
                        '8.0.42-33': [
                            mysql:     "${S3_BASE}/mysql/Percona-Server-8.0.42-33-Linux.x86_64.glibc2.28.tar.gz",
                            xtrabackup:"${S3_BASE}/xtrabackup/percona-xtrabackup-8.0.35-34-Linux-x86_64.glibc2.28.tar.gz",
                            shell:     "${S3_BASE}/mysql-shell/mysql-shell-8.0.43-linux-glibc2.28-x86-64bit.tar.gz",
                            toolkit:   "${S3_BASE}/percona-toolkit/percona-toolkit-3.7.0-2_x86_64.tar.gz",
                            proxysql:  "${S3_BASE}/proxysql/proxysql-2.7.3-Linux-x86_64.glibc2.28.tar.gz"
                        ],
                        '8.0.40-31': [
                            mysql:     "${S3_BASE}/mysql/Percona-Server-8.0.40-31-Linux.x86_64.glibc2.28.tar.gz",
                            xtrabackup:"${S3_BASE}/xtrabackup/percona-xtrabackup-8.0.33-28-Linux-x86_64.glibc2.28.tar.gz",
                            shell:     "${S3_BASE}/mysql-shell/mysql-shell-8.0.41-linux-glibc2.28-x86-64bit.tar.gz",
                            toolkit:   "${S3_BASE}/percona-toolkit/percona-toolkit-3.6.0_x86_64.tar.gz",
                            proxysql:  "${S3_BASE}/proxysql/proxysql-2.6.1-Linux-x86_64.glibc2.28.tar.gz"
                        ],
                        '8.4.4-4': [
                            mysql:     "${S3_BASE}/mysql/Percona-Server-8.4.4-4-Linux.x86_64.glibc2.28.tar.gz",
                            xtrabackup:"${S3_BASE}/xtrabackup/percona-xtrabackup-8.4.0-2-Linux-x86_64.glibc2.28.tar.gz",
                            shell:     "${S3_BASE}/mysql-shell/mysql-shell-8.4.4-linux-glibc2.28-x86-64bit.tar.gz",
                            toolkit:   "${S3_BASE}/percona-toolkit/percona-toolkit-3.7.0-2_x86_64.tar.gz",
                            proxysql:  "${S3_BASE}/proxysql/proxysql-2.7.3-Linux.x86_64.glibc2.28.tar.gz"
                        ]
                    ]
                    def chosen = s3Map[params.MYSQL_MINOR]
                    if (!chosen) { error("No S3 path mapping found for MYSQL_MINOR=${params.MYSQL_MINOR}") }

                    env.S3_MYSQL     = chosen.mysql
                    env.S3_XTRABACKUP= chosen.xtrabackup
                    env.S3_SHELL     = chosen.shell
                    env.S3_TOOLKIT   = chosen.toolkit
                    env.S3_PROXYSQL  = chosen.proxysql

                    echo "📦 Resolved S3 paths for version ${params.MYSQL_MINOR}"
                }
            }
        }

        // ----------------------------------------------------------
        stage('Generate Inventory & Vars') {
        // ----------------------------------------------------------
            steps {
                script {
                    def dirBase = "${params.MYSQL_DATA_DIR}/${params.APP_NAME}/${params.ENV_NAME}"
                    def uuid    = sh(script: "uuidgen", returnStdout: true).trim()

                    // ── hosts.ini ──────────────────────────────
                    def hostsContent = """[primary]
${params.NODE1_IP}

[secondary]
${params.NODE2_IP}
${params.NODE3_IP}

[cluster:children]
primary
secondary

[cluster:vars]
ansible_user=mysql
ansible_ssh_private_key_file=~/.ssh/id_rsa
ansible_become=true
"""
                    writeFile file: env.INVENTORY_FILE, text: hostsContent

                    // ── vars.yml ───────────────────────────────
                    def varsContent = """---
mysql_version: "${params.MYSQL_MINOR}"
mysql_download_url: "${env.S3_MYSQL}"
perona_xtrabackup_download_url: "${env.S3_XTRABACKUP}"
percona_toolkit_download_url: "${env.S3_TOOLKIT}"
proxysql_download_url: "${env.S3_PROXYSQL}"
mysql_shell_download_url: "${env.S3_SHELL}"

mysql_port: ${params.MYSQL_PORT}
mysql_groupreplication_port: ${params.MYSQL_GR_PORT}
mysql_report_port: ${params.MYSQL_PORT}

mysql_app: "${params.APP_NAME}"
mysql_env: "${params.ENV_NAME}"
mysql_dc: "${params.DC_NAME}"

mysql_directories_location: "${dirBase}"
mysql_basedir_location: "${dirBase}/binaries/innodbcluster-${params.MYSQL_MINOR}"
percona_xtrabackup_basedir_location: "${dirBase}/binaries/percona-xtrabackup"
percona_toolkit_basedir_location: "${dirBase}/binaries/percona-toolkit"
mysql_shell_basedir_location: "${dirBase}/binaries/mysql-shell"
proxysql_basedir_location: "${dirBase}/binaries/proxysql"

mysql_conf_location: "${dirBase}/conf"
mysql_datadir_location: "${dirBase}/data"
mysql_logdir_location: "${dirBase}/logs"
mysql_tempdir_location: "${dirBase}/temp"
mysql_binlogdir_location: "${dirBase}/binlogs"
mysql_relaylogdir_location: "${dirBase}/relaylogs"
mysql_ssl_location: "${dirBase}/ssl"

mysql_socket: "${dirBase}/data/mysql.sock"
mysql_pid_file: "${dirBase}/data/mysqld.pid"
mysql_log_error: "${dirBase}/logs/error.log"
mysql_slow_query_log_file: "${dirBase}/logs/slow.log"
mysql_general_log_file: "${dirBase}/logs/general.log"
mysql_log_bin: "${dirBase}/binlogs/mysql-bin"
mysql_log_bin_index: "${dirBase}/binlogs/mysql-bin.index"
mysql_relay_log: "${dirBase}/relaylogs/relay-log"
mysql_relay_log_index: "${dirBase}/relaylogs/relay-log.index"
mysql_innodb_data_home_dir: "${dirBase}/data"
mysql_innodb_log_group_home_dir: "${dirBase}/logs"

mysql_super_user: "root"
mysql_super_password: "{{ vault_mysql_super_password }}"
mysql_cluster_admin_user: "clusteradmin"
mysql_cluster_admin_user_password: "{{ vault_cluster_admin_password }}"

mysql_max_connections: 500
mysql_max_user_connections: 490
mysql_max_allowed_packet: "256M"
mysql_sort_buffer_size: "4M"
mysql_innodb_buffer_pool_size: "4G"
mysql_innodb_log_buffer_size: "64M"
mysql_innodb_log_file_size: "512M"
mysql_lower_case_table_names: 1
mysql_long_query_time: 2

mysql_ssl_ca: "${dirBase}/ssl/ca.pem"
mysql_ssl_cert: "${dirBase}/ssl/server-cert.pem"
mysql_ssl_key: "${dirBase}/ssl/server-key.pem"

mysql_group_replication_group_name: "${uuid}"
"""
                    writeFile file: env.VARS_FILE, text: varsContent
                    echo "📝 Inventory and vars.yml generated"
                }
            }
        }

        // ----------------------------------------------------------
        stage('Dry Run (--check)') {
        // ----------------------------------------------------------
            when { expression { params.DRY_RUN == true } }
            steps {
                withCredentials([
                    sshUserPrivateKey(credentialsId: params.SSH_CRED_ID, keyFileVariable: 'SSH_KEY'),
                    string(credentialsId: params.SUDO_PASS_CRED, variable: 'SUDO_PASS')
                ]) {
                    script {
                        def tag = params.ANSIBLE_TAGS == 'all' ? '' : "--tags \"${params.ANSIBLE_TAGS}\""
                        sh """
                            export ANSIBLE_HOST_KEY_CHECKING=False
                            ansible-playbook -i ${env.INVENTORY_FILE} ${env.PLAYBOOK_MAIN} \\
                                -e @${env.VARS_FILE} \\
                                --private-key=\$SSH_KEY \\
                                --extra-vars "ansible_become_pass=\$SUDO_PASS" \\
                                ${tag} --check
                        """
                    }
                }
            }
        }

        // ----------------------------------------------------------
        stage('Deploy — Install Dependencies') {
        // ----------------------------------------------------------
            when {
                expression { params.DRY_RUN == false }
                expression { params.ANSIBLE_TAGS in ['all','install_dependencies'] }
            }
            steps { ansibleStage('install_dependencies') }
        }

        // ----------------------------------------------------------
        stage('Deploy — Download Binaries from S3') {
        // ----------------------------------------------------------
            when {
                expression { params.DRY_RUN == false }
                expression { params.ANSIBLE_TAGS in ['all','download_binaries'] }
            }
            steps { ansibleStage('download_binaries') }
        }

        // ----------------------------------------------------------
        stage('Deploy — Install MySQL') {
        // ----------------------------------------------------------
            when {
                expression { params.DRY_RUN == false }
                expression { params.ANSIBLE_TAGS in ['all','install_mysql'] }
            }
            steps { ansibleStage('install_mysql') }
        }

        // ----------------------------------------------------------
        stage('Deploy — Configure Group Replication') {
        // ----------------------------------------------------------
            when {
                expression { params.DRY_RUN == false }
                expression { params.ANSIBLE_TAGS in ['all','setup_group_replication'] }
            }
            steps { ansibleStage('setup_group_replication') }
        }

        // ----------------------------------------------------------
        stage('Deploy — Configure InnoDB Cluster') {
        // ----------------------------------------------------------
            when {
                expression { params.DRY_RUN == false }
                expression { params.ANSIBLE_TAGS in ['all','configure_cluster'] }
            }
            steps { ansibleStage('configure_cluster') }
        }

        // ----------------------------------------------------------
        stage('Validate Cluster') {
        // ----------------------------------------------------------
            when { expression { params.DRY_RUN == false } }
            steps {
                withCredentials([
                    sshUserPrivateKey(credentialsId: params.SSH_CRED_ID, keyFileVariable: 'SSH_KEY'),
                    string(credentialsId: params.SUDO_PASS_CRED, variable: 'SUDO_PASS')
                ]) {
                    sh """
                        export ANSIBLE_HOST_KEY_CHECKING=False
                        ansible-playbook -i ${env.INVENTORY_FILE} ${WORK_DIR}/playbooks/validate_setup.yml \\
                            -e @${env.VARS_FILE} \\
                            --private-key=\$SSH_KEY \\
                            --extra-vars "ansible_become_pass=\$SUDO_PASS"
                    """
                }
            }
        }
    }

    post {
        success { echo "✅ MySQL InnoDB Cluster deployment completed successfully." }
        failure { echo "❌ Deployment failed. Check console output above." }
        always  { archiveArtifacts artifacts: 'ansible/inventory/**', allowEmptyArchive: true }
    }
}

// ── Helper function to avoid repeating withCredentials blocks ──
def ansibleStage(String tag) {
    withCredentials([
        sshUserPrivateKey(credentialsId: params.SSH_CRED_ID, keyFileVariable: 'SSH_KEY'),
        string(credentialsId: params.SUDO_PASS_CRED, variable: 'SUDO_PASS')
    ]) {
        sh """
            export ANSIBLE_HOST_KEY_CHECKING=False
            ansible-playbook -i ${env.INVENTORY_FILE} ${env.PLAYBOOK_MAIN} \\
                -e @${env.VARS_FILE} \\
                --private-key=\$SSH_KEY \\
                --extra-vars "ansible_become_pass=\$SUDO_PASS" \\
                --tags "${tag}"
        """
    }
}
