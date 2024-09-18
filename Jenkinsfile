// example Jenkinsfile for Black Duck scans using the Synopsys Security Scan Plugin
// https://plugins.jenkins.io/synopsys-security-scan
pipeline {
    agent { label 'linux64' }
    environment {
        REPO_NAME = "${env.GIT_URL.tokenize('/.')[-2]}"
        FULLSCAN = "${env.BRANCH_NAME ==~ /^(main|master|develop|stage|release)$/ ? 'true' : 'false'}"
        PRSCAN = "${env.CHANGE_TARGET ==~ /^(main|master|develop|stage|release)$/ ? 'true' : 'false'}"
        BRIDGECLI_LINUX64 = 'https://sig-repo.synopsys.com/artifactory/bds-integrations-release/com/synopsys/integration/synopsys-bridge/latest/synopsys-bridge-linux64.zip'
        BRIDGE_BLACKDUCK_URL = 'https://poc357.blackduck.synopsys.com'
        BRIDGE_BLACKDUCK_URL = credentials('poc357.blackduck.synopsys.com')
        DETECT_PROJECT_NAME = "${env.REPO_NAME}"
        GITHUB_TOKEN = credentials('github-pat')
    }
    tools {
        maven 'maven-3.9'
        jdk 'openjdk-17'
    }
    stages {
        stage('Environment') {
            steps {
                sh 'env | sort'
            }
        }
        stage('Build') {
            steps {
                sh 'mvn -B package'
            }
        }
/*
        stage('Black Duck') {
            steps {
                synopsys_scan product: 'blackduck',
                    blackduck_scan_failure_severities: 'BLOCKER',
                    blackduck_reports_sarif_create: true,
                    blackduck_automation_prcomment: true,
                    mark_build_status: 'UNSTABLE',
                    github_token: "$GITHUB_TOKEN"
            }
        }
*/
        stage('Black Duck Full Scan') {
            when { environment name: 'FULLSCAN', value: 'true' }
            steps {
                script {
                    status = sh returnStatus: true, script: '''
                        curl -fLsS -o bridge.zip $BRIDGECLI_LINUX64 && unzip -qo -d $WORKSPACE_TMP bridge.zip && rm -f bridge.zip
                        $WORKSPACE_TMP/synopsys-bridge --verbose --stage blackduck \
                            blackduck.scan.full=true \
                            blackduck.scan.failure.severities='BLOCKER' \
                            blackduck.fixpr.enabled=true \
                            blackduck.reports.sarif.create=true \
                            github.repository.name=$REPO_NAME \
                            github.repository.branch.name=$BRANCH_NAME \
                            github.repository.owner.name=chuckaude-org \
                            github.user.token=$GITHUB_TOKEN
                    '''
                    if (status == 8) { unstable 'policy violation' }
                    else if (status != 0) { error 'scan failure' }
                }
            }
        }
        stage('Black Duck PR Scan') {
            when { environment name: 'PRSCAN', value: 'true' }
            steps {
                script {
                    status = sh returnStatus: true, script: '''
                        curl -fLsS -o bridge.zip $BRIDGECLI_LINUX64 && unzip -qo -d $WORKSPACE_TMP bridge.zip && rm -f bridge.zip
                        $WORKSPACE_TMP/synopsys-bridge --verbose --stage blackduck \
                            blackduck.scan.full=false \
                            blackduck.automation.prcomment=true \
                            github.repository.name=$REPO_NAME \
                            github.repository.branch.name=$BRANCH_NAME \
                            github.repository.owner.name=chuckaude-org \
                            github.repository.pull.number=$CHANGE_ID \
                            github.user.token=$GITHUB_TOKEN
                    '''
                    if (status == 8) { unstable 'policy violation' }
                    else if (status != 0) { error 'scan failure' }
                }
            }
        }
    }
    post {
        always {
            archiveArtifacts allowEmptyArchive: true, artifacts: '.bridge/bridge.log, .bridge/*/report.sarif.json'
            //zip archive: true, dir: '.bridge', zipFile: 'bridge-logs.zip'
            cleanWs()
        }
    }
}
