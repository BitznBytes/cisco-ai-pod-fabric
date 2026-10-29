// AI Pod — Network as Code pipeline
// Manages three fabrics: FEN (VXLAN EVPN), BEN (AI/GPU routed), Admin (OOB Classic LAN)
//
// Stage execution by branch:
//   All branches : Setup, Validate
//   main only    : Provision Non-NAC Fabrics, Deploy, Configure Extended,
//                  Wait Managed, Test, Workspace Cleanup
//
// Jenkins credentials required (all "Secret text" type):
//   ND_HOST, ND_USERNAME, ND_PASSWORD       — Nexus Dashboard access
//   NDFC_SW_USERNAME, NDFC_SW_PASSWORD      — switch credentials for NDFC
//   webex-chat-ops-credentials              — Webex bot token for notifications

pipeline {
    options {
        disableConcurrentBuilds()
        ansiColor('css')
    }

    agent {
        // CUSTOMIZE: Update to match your Jenkins build agent label
        label 'YOUR-BUILD-AGENT-LABEL'
    }

    environment {
        // Nexus Dashboard credentials
        // Must be "Secret text" type — "Username with password" sets <ID>_USR/<ID>_PSW,
        // not the bare env var name that Ansible and the Python scripts expect.
        // ND_HOST: IP or hostname only, no https://
        ND_HOST      = credentials('NDFC_HOST')
        ND_USERNAME  = credentials('ND_USERNAME')
        ND_PASSWORD  = credentials('ND_PASSWORD')
        ND_DOMAIN    = "local"
        // ND_FQDN = credentials('ND_FQDN')  // uncomment if cert uses FQDN but ND_HOST is an IP

        // Switch credentials (used by NDFC when configuring devices,
        // and by ndfc_provision_fabrics.py --discover-switches)
        NDFC_SW_USERNAME = credentials('NDFC_SW_USERNAME')
        NDFC_SW_PASSWORD = credentials('NDFC_SW_PASSWORD')

        // Webex notification settings
        WEBEX_ROOM_ID      = "YOUR-WEBEX-ROOM-ID"   // CUSTOMIZE: Webex space ID for notifications
        GIT_COMMIT         = "${sh(returnStdout: true, script: 'git rev-parse HEAD').trim()}"
        GIT_COMMIT_MESSAGE = "${sh(returnStdout: true, script: 'git log -1 --pretty=%B ${GIT_COMMIT}').trim()}"
        GIT_AUTHOR         = "${sh(returnStdout: true, script: 'git log -1 --pretty=%an ${GIT_COMMIT}').trim()}"
        GIT_EVENT          = "${sh(returnStdout: true, script: 'echo $GIT_EVENT').trim()}"
    }

    stages {

        stage('Start Notification') {
            steps {
                script { sendWebexStartNotification() }
            }
        }

        // ── SETUP ─────────────────────────────────────────────────────────────
        stage('Setup') {
            steps {
                sh '''
                    python3 --version
                    python3 -m venv venv
                    . venv/bin/activate
                    pip3 install -r requirements.txt 2>&1 | tee pip_install_output.txt
                '''
                sh '''
                    . venv/bin/activate
                    ansible-galaxy collection install -r requirements.yaml -f 2>&1 | tee ansible_galaxy_output.txt
                '''
                sh '''
                    . venv/bin/activate
                    ansible --version
                    nac-validate --version
                    nac-test --version
                '''
            }
            post {
                failure {
                    script {
                        def out = fileExists('pip_install_output.txt') ? readFile('pip_install_output.txt').trim()
                                : fileExists('ansible_galaxy_output.txt') ? readFile('ansible_galaxy_output.txt').trim() : ''
                        sendWebexFailureNotification('Setup', out)
                    }
                }
            }
        }

        // ── VALIDATE ──────────────────────────────────────────────────────────
        // Runs on every branch — catches data model errors before any NDFC changes.
        // Only validates FEN (*.nac.yaml files); BEN and Admin use fabric_provision.yaml
        // which is outside the nac-validate schema scope.
        stage('Validate') {
            steps {
                sh '''#!/bin/bash
                    . venv/bin/activate
                    set -o pipefail
                    nac-validate host_vars/ --verbosity DEBUG 2>&1 | tee nac_validate_output.txt
                '''
                archiveArtifacts 'nac_validate_output.txt'
                sh '''#!/bin/bash
                    . venv/bin/activate
                    set -o pipefail
                    ansible-playbook -i inventory.yaml validate.yaml 2>&1 | tee ansible_validate_output.txt
                '''
                archiveArtifacts 'ansible_validate_output.txt'
            }
            post {
                failure {
                    script {
                        def out = fileExists('nac_validate_output.txt') ? readFile('nac_validate_output.txt').trim()
                                : fileExists('ansible_validate_output.txt') ? readFile('ansible_validate_output.txt').trim() : ''
                        sendWebexFailureNotification('Validate', out)
                    }
                }
            }
        }

        // ── PROVISION NON-NAC FABRICS ─────────────────────────────────────────
        // Creates or updates BEN (Easy_Fabric_eBGP) and Admin (Easy_Fabric_Classic)
        // in NDFC via REST API. Idempotent — safe to run on every deployment.
        //
        // NOTE: This stage requires ndfc_provision_fabrics.py which is NOT included
        // in this public template. Implement your own NDFC provisioning script based
        // on the fabric definitions in host_vars/BEN/fabric_provision.yaml and
        // host_vars/Admin/fabric_provision.yaml.
        stage('Provision Non-NAC Fabrics') {
            when { branch 'main' }
            steps {
                sh '''#!/bin/bash
                    echo "ERROR: This stage requires ndfc_provision_fabrics.py which is not included in this public template."
                    echo "Implement your own NDFC provisioning script for BEN and Admin fabrics."
                    echo "See host_vars/BEN/fabric_provision.yaml and host_vars/Admin/fabric_provision.yaml for the fabric definitions."
                    exit 1
                '''
            }
            post {
                failure {
                    script {
                        sendWebexFailureNotification('Provision Non-NAC Fabrics', 'Script not implemented — see stage comments.')
                    }
                }
            }
        }

        // ── DEPLOY ────────────────────────────────────────────────────────────
        // Runs cisco.nac_dc_vxlan.dtc.create then .dtc.deploy for FEN.
        // BEN and Admin are handled by ndfc_provision_fabrics.py above.
        stage('Deploy') {
            when { branch 'main' }
            steps {
                sh '''#!/bin/bash
                    . venv/bin/activate
                    set -o pipefail
                    ansible-playbook -i inventory.yaml vxlan.yaml 2>&1 | tee ansible_deploy_output.txt
                '''
                archiveArtifacts 'ansible_deploy_output.txt'
            }
            post {
                failure {
                    script {
                        def out = fileExists('ansible_deploy_output.txt') ? readFile('ansible_deploy_output.txt').trim() : ''
                        sendWebexFailureNotification('Deploy', out)
                    }
                }
            }
        }

        // ── CONFIGURE EXTENDED ────────────────────────────────────────────────
        // Applies nvPairs not exposed by the collection (CDP, NGOAM, etc.)
        // to FEN via the NDFC REST API.
        //
        // NOTE: This stage requires ndfc_apply_fabric_ext.py which is NOT included
        // in this public template. The extended settings to apply are defined in
        // host_vars/FEN/fabric_ext.nac.yaml. Implement your own script to POST
        // those nvPairs to the NDFC fabric REST API.
        stage('Configure Extended') {
            when { branch 'main' }
            steps {
                sh '''#!/bin/bash
                    echo "ERROR: This stage requires ndfc_apply_fabric_ext.py which is not included in this public template."
                    echo "Extended fabric settings are defined in host_vars/FEN/fabric_ext.nac.yaml."
                    echo "Implement your own NDFC REST API script to apply them, or apply manually via the NDFC GUI."
                    exit 1
                '''
            }
            post {
                failure {
                    script {
                        sendWebexFailureNotification('Configure Extended', 'Script not implemented — see stage comments.')
                    }
                }
            }
        }

        // ── WAIT MANAGED ──────────────────────────────────────────────────────
        // Polls NDFC until all FEN switches reach In-Sync / managedMode state.
        // Gates the Test stage — tests against unmanaged switches will fail.
        //
        // NOTE: This stage requires ndfc_wait_managed.py which is NOT included
        // in this public template. Implement your own polling script using the
        // NDFC REST API endpoint: GET /appcenter/cisco/ndfc/api/v1/lan-fabric/rest/control/fabrics/{fabric}/inventory/manageable
        stage('Wait Managed') {
            when { branch 'main' }
            steps {
                sh '''#!/bin/bash
                    echo "ERROR: This stage requires ndfc_wait_managed.py which is not included in this public template."
                    echo "Implement your own NDFC switch managed-state polling script."
                    exit 1
                '''
            }
            post {
                failure {
                    script {
                        sendWebexFailureNotification('Wait Managed', 'Script not implemented — see stage comments.')
                    }
                }
            }
        }

        // ── TEST ──────────────────────────────────────────────────────────────
        // Runs nac-test against FEN. BEN and Admin do not have nac-test templates yet.
        stage('Test') {
            when { branch 'main' }
            steps {
                sh '''#!/bin/bash
                    . venv/bin/activate
                    set -o pipefail
                    > test_output.txt
                    for fabric_dir in host_vars/*/; do
                        fabric=$(basename "$fabric_dir")
                        # Only run nac-test against collection-managed fabrics (*.nac.yaml present)
                        if ls "$fabric_dir"*.nac.yaml > /dev/null 2>&1; then
                            echo "=== Testing fabric: $fabric ===" | tee -a test_output.txt
                            nac-test \
                                -d "$fabric_dir" \
                                -d defaults \
                                -t tests/templates \
                                -f jinja_filters \
                                -o "tests/results/vxlan/${fabric}" \
                                2>&1 | tee -a test_output.txt
                        else
                            echo "=== Skipping fabric: $fabric (no *.nac.yaml — not collection-managed) ===" | tee -a test_output.txt
                        fi
                    done
                '''
                archiveArtifacts artifacts: 'test_output.txt', allowEmptyArchive: true
                archiveArtifacts artifacts: 'tests/results/vxlan/**', allowEmptyArchive: true
            }
            post {
                failure {
                    script {
                        def out = fileExists('test_output.txt') ? readFile('test_output.txt').trim() : ''
                        sendWebexFailureNotification('Test', out)
                    }
                }
            }
        }

        // ── WORKSPACE CLEANUP ─────────────────────────────────────────────────
        // Only runs on success — keep workspace on failure for debugging.
        stage('Workspace Cleanup') {
            steps {
                cleanWs(
                    cleanWhenNotBuilt: false,
                    deleteDirs: true,
                    disableDeferredWipeout: true,
                    notFailBuild: true,
                    patterns: [[pattern: '.propsfile', type: 'EXCLUDE']]
                )
            }
        }
    }

    post {
        success {
            script { sendWebexSuccessNotification() }
        }
    }
}

// ── NOTIFICATION HELPERS ───────────────────────────────────────────────────────

def sendWebexFailureNotification(String failedStage, String errorOutput = "") {
    def commit       = env.GIT_COMMIT ?: 'Unknown'
    def commitAuthor = env.GIT_AUTHOR ?: 'Unknown'
    def commitMsg    = env.GIT_COMMIT_MESSAGE ?: 'No commit message'
    def branch       = env.BRANCH_NAME ?: 'Unknown'
    def duration     = currentBuild.durationString ?: 'Unknown'

    def stageEmoji = [
        'Setup'                    : '⚙️',
        'Validate'                 : '🔍',
        'Provision Non-NAC Fabrics': '🏗️',
        'Deploy'                   : '🚀',
        'Configure Extended'       : '🔧',
        'Wait Managed'             : '⏳',
        'Test'                     : '🧪',
    ][failedStage] ?: '❌'

    def truncatedOutput = errorOutput
    if (errorOutput?.trim()) {
        def lines = errorOutput.split('\n')
        if (lines.length > 50) {
            truncatedOutput = "... (showing last 50 lines)\n" + lines[-50..-1].join('\n')
        }
    }

    def message = """
## 💀 AI Pod Network Infrastructure Failure 💀
### ${stageEmoji} Failed Stage: **${failedStage}**

---
**📊 Build Information:**
* **Pipeline**: ${JOB_NAME}
* **Branch**: `${branch}`
* **Duration**: ${duration}
* **Build**: [${BUILD_DISPLAY_NAME}](${env.BUILD_URL})

**📝 Git Information:**
* **Commit**: `${commit.take(8)}...`
* **Author**: ${commitAuthor}
* **Message**: _${commitMsg}_

**🔗 Quick Links:**
* [📋 Build Details](${env.BUILD_URL})
* [📜 Console Output](${env.BUILD_URL}console)"""

    if (truncatedOutput?.trim()) {
        message += """

---
### 🚨 Error Details:
```
${truncatedOutput}
```"""
    }

    sparkSend credentialsId: 'webex-chat-ops-credentials',
              message: message,
              messageType: 'markdown',
              spaceList: [[spaceId: "${env.WEBEX_ROOM_ID}"]]
}

def sendWebexStartNotification() {
    def commit    = env.GIT_COMMIT ?: 'Unknown'
    def author    = env.GIT_AUTHOR ?: 'Unknown'
    def commitMsg = env.GIT_COMMIT_MESSAGE ?: 'No commit message'
    def branch    = env.BRANCH_NAME ?: 'Unknown'
    def timestamp = new Date().format("yyyy-MM-dd HH:mm:ss 'UTC'", TimeZone.getTimeZone('UTC'))

    def deployContext = branch == 'main' ?
        '🔴 **PRODUCTION** — Full pipeline: Provision → Deploy → Configure → Wait → Test' :
        '🟡 **VALIDATION ONLY** — Setup + Validate only'

    sparkSend credentialsId: 'webex-chat-ops-credentials',
    message: ("""
## 🌟 AI Pod Network Infrastructure Pipeline Started 🌟

---
**📊 Pipeline Information:**
* **Pipeline**: ${JOB_NAME}
* **Branch**: `${branch}`
* **Build**: [${BUILD_DISPLAY_NAME}](${env.BUILD_URL})
* **Started**: ${timestamp}

**📝 Git Information:**
* **Commit**: `${commit.take(8)}...`
* **Author**: ${author}
* **Message**: _${commitMsg}_

**🎯 Execution Context:**
${deployContext}

**🔧 Pipeline Stages (main):**
* ⚙️ Setup → 🔍 Validate → 🏗️ Provision Non-NAC Fabrics → 🚀 Deploy → 🔧 Configure Extended → ⏳ Wait Managed → 🧪 Test

**🔗 Quick Links:**
* [📋 Build Details](${env.BUILD_URL})
* [📜 Live Console](${env.BUILD_URL}console)
    """),
    messageType: 'markdown',
    spaceList: [[spaceId: "${env.WEBEX_ROOM_ID}"]]
}

def sendWebexSuccessNotification() {
    def commit    = env.GIT_COMMIT ?: 'Unknown'
    def author    = env.GIT_AUTHOR ?: 'Unknown'
    def commitMsg = env.GIT_COMMIT_MESSAGE ?: 'No commit message'
    def branch    = env.BRANCH_NAME ?: 'Unknown'
    def duration  = currentBuild.durationString ?: 'Unknown'
    def timestamp = new Date().format("yyyy-MM-dd HH:mm:ss 'UTC'", TimeZone.getTimeZone('UTC'))

    def accomplishments = branch == 'main' ?
        """* ✅ BEN + Admin fabrics provisioned/verified in NDFC
* ✅ FEN deployed via cisco.nac_dc_vxlan collection
* ✅ Extended fabric settings applied (CDP, NGOAM)
* ✅ All switches reached managed state
* ✅ nac-test passed""" :
        """* ✅ Data model validated against schema
* ✅ Ready for deployment on main"""

    sparkSend credentialsId: 'webex-chat-ops-credentials',
    message: ("""
## 🎉 AI Pod Network Infrastructure Pipeline Successful! 🎉

---
**📊 Build Summary:**
* **Pipeline**: ${JOB_NAME}
* **Branch**: `${branch}`
* **Duration**: ${duration}
* **Completed**: ${timestamp}
* **Build**: [${BUILD_DISPLAY_NAME}](${env.BUILD_URL})

**📝 Git Information:**
* **Commit**: `${commit.take(8)}...`
* **Author**: ${author}
* **Message**: _${commitMsg}_

**🏆 Accomplishments:**
${accomplishments}

**🔗 Quick Links:**
* [📋 Build Details](${env.BUILD_URL})
* [📜 Console Output](${env.BUILD_URL}console)
    """),
    messageType: 'markdown',
    spaceList: [[spaceId: "${env.WEBEX_ROOM_ID}"]]
}
