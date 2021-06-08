pipeline {
  agent any
  stages {
    stage ('Prepare fuzz test') {
      environment {
        // Name of the project to fuzz
        PROJECT_NAME = 'projects/73848196_github_webgoat-cicd-testing-0c6c30b1'
        // Address of the fuzzing service
        FUZZING_SERVER_URL = 'server-installer-test.code-intelligence.com:6773'
        // Address of the fuzzing web interface
        WEB_APP_ADDRESS =  'https://server-installer-test.code-intelligence.com'

        // The git branch to use for the fuzz test
        GIT_BRANCH = 'master'

        // Credentials for accessing the fuzzing service
        CI_FUZZ_API_TOKEN = credentials('CI_FUZZ_API_TOKEN')
        CICTL = "${WORKSPACE}/cictl-3.1.1-linux";
        CICTL_VERSION = 'cictl-3.1.1';
        CICTL_SHA256SUM = '96118aa6a89a8a2dfe325ec204f1276d6c4144ecc66f8f0a6f38e359a20a1152';
        CICTL_URL = 'https://s3.eu-central-1.amazonaws.com/public.code-intelligence.com/cictl/cictl-3.1.1-linux';
        FINDINGS_TYPE = 'CRASH';
        TIMEOUT = '900'
  
      }

      stages {
        stage ('Deploy SUT') {
          steps {
            sh '''
              mvn clean install -DskipTests
              java -javaagent:$HOME/bin/fuzzing_agent_deploy.jar=instrumentation_includes="org.owasp.**",service_name=projects/webgoat-7ccae1b8/web_services/webgoat,fuzzing_server_host=server-installer-test.code-intelligence.com -jar ./webgoat-server/target/webgoat-server-8.0.0-SNAPSHOT.jar &
            '''
          }
        }
        
        stage ('Download cictl') {
          steps {
            sh '''
              set -eu

              # Download cictl if it doesn't exist already
              if [ ! -f "${CICTL}" ]; then
                curl "${CICTL_URL}" -o "${CICTL}"
              fi

              # Verify the checksum
              echo "${CICTL_SHA256SUM} "${CICTL}"" | sha256sum --check

              # Make it executable
              chmod +x "${CICTL}"
            '''
          }
        }

        stage ('Build fuzz test') {
          steps {
            sh '''
              set -eu

              # Switch to build directory
              mkdir -p "${BUILD_TAG}"
              cd "${BUILD_TAG}"

              # Log in
              echo "${CI_FUZZ_API_TOKEN}" | $CICTL --server="${FUZZING_SERVER_URL}" login --quiet

              # $CI_COMMIT_SHA may be specified in the jenkins pipeline,
              # or, if using the Git plugin, $GIT_COMMIT could be used.
              if [ -z "${CI_COMMIT_SHA:-}" ]; then
                CI_COMMIT_SHA=${GIT_COMMIT:-}
              fi
              
              # Start fuzzing.
              CAMPAIGN_RUN=$(${CICTL} start \\
		-f=projects/73848196_github_webgoat-cicd-testing-0c6c30b1/fuzz_targets/LmNvZGUtaW50ZWxsaWdlbmNlL2Z1enpfdGFyZ2V0cy9mdXp6X3Rlc3RfMQ== \\
           	--server="${FUZZING_SERVER_URL}" \\
                --report-email="${REPORT_EMAIL:-}" \\
                --git-branch="${GIT_BRANCH}" \\
                --commit-sha="${CI_COMMIT_SHA:-}" \\
                "${PROJECT_NAME}")

              # Store the campaign run name for the next stage
              OUTFILE="campaign-run"
              echo "${CAMPAIGN_RUN}" > "${OUTFILE}"
            '''
          }
        }

        stage ('Start fuzz test') {
          steps {
            sh '''
              set -eu

              # Switch to build directory
              cd "${BUILD_TAG}"

              # Get the name of the started campaign run
              INFILE="campaign-run"
              CAMPAIGN_RUN=$(cat ${INFILE})

              # Log in
              echo "${CI_FUZZ_API_TOKEN}" | ${CICTL} --server="${FUZZING_SERVER_URL}" login --quiet

              # Monitor Fuzzing
              ${CICTL} monitor_campaign_run \\
                --server="${FUZZING_SERVER_URL}" \\
                --dashboard_address="${WEB_APP_ADDRESS}" \\
                --duration="${TIMEOUT}" \\
                --findings_type="${FINDINGS_TYPE}" \\
                "${CAMPAIGN_RUN}"
            '''
          }
        }
      }
    }
  }

  post {
    always {
        sh '''
        set -eu

        # Switch to build directory
        cd "${BUILD_TAG}"

        # Check if there are any findings
        if ! stat -t finding-*.json > /dev/null 2>&1; then
          # There are no findings, so there's nothing to do
          exit
        fi

        JQ="${WORKSPACE}/jq"
        JQ_URL=https://github.com/stedolan/jq/releases/download/jq-1.6/jq-linux64
        JQ_CHECKSUM=af986793a515d500ab2d35f8d2aecd656e764504b789b66d7e1a0b727a124c44

        # Download jq if it doesn't exist
        if [ ! -f "${JQ}" ]; then
          curl "${JQ_URL}" -o "${JQ}"
        fi

        # Verify the checksum
        echo "${JQ_CHECKSUM}" "${JQ}" | sha256sum --check

        # Make it executable
        chmod +x "${JQ}"

        # Merge findings into one file
        "${JQ}" --slurp '.' finding-*.json > cifuzz_findings.json
        '''

        archiveArtifacts artifacts: "${BUILD_TAG}/cifuzz_findings.json", fingerprint: true
    }
  }
}
