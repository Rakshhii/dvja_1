pipeline {
    agent any

    environment {
        CX_CLI_PATH = "C:\\Rakshii\\cx-cli\\cx.exe"
    }

    stages {
        stage('Checkout SCM') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/1.1']],
                    userRemoteConfigs: [[
                        url: 'https://github.com/Rakshhii/dvja_1',
                        credentialsId: 'rakshi'
                    ]]
                ])
            }
        }

        stage('Configure CxOne CLI') {
            steps {
                withCredentials([string(credentialsId: 'cx-api-key', variable: 'CX_APIKEY')]) {
                    bat """
                        echo Configuring CxOne CLI...
                        ${env.CX_CLI_PATH} configure set --prop-name cx_base_uri --prop-value https://ind.ast.checkmarx.net
                        ${env.CX_CLI_PATH} configure set --prop-name cx_tenant --prop-value cx_ind_internal_test
                        echo API key will be passed via environment variable.
                    """
                }
            }
        }

        stage('Run SAST + SCA + Container Security Scans') {
            steps {
                withCredentials([string(credentialsId: 'cx-api-key', variable: 'CX_APIKEY')]) {
                    script {
                        echo "Running SAST + SCA + Container Security scans..."
                        def scanOutput = bat(
                            script: """
                                set CX_APIKEY=%CX_APIKEY%
                                ${env.CX_CLI_PATH} scan create ^
                                    --project-name "Rakshhii/dvja_1" ^
                                    --branch "master" ^
                                    -s . ^
                                    --scan-types sast,sca,container-security ^
                                    --container-images goat:ex1
                            """,
                            returnStdout: true
                        ).trim()
                        echo "Scan Output:\n${scanOutput}"

                        // Extract Scan ID from output
                        def scanIdMatcher = scanOutput =~ /Scan ID\s*:\s*([a-f0-9\\-]+)/
                        if (scanIdMatcher) {
                            env.SCAN_ID = scanIdMatcher[0][1]
                            echo "Extracted Scan ID: ${env.SCAN_ID}"
                        } else {
                            error("Failed to extract Scan ID from scan output.")
                        }
                    }
                }
            }
        }

        stage('Check Results / Quality Gate') {
            steps {
                withCredentials([string(credentialsId: 'cx-api-key', variable: 'CX_APIKEY')]) {
                    script {
                        echo "Fetching results for Scan ID: ${env.SCAN_ID}"

                        def resultOutput = bat(
                            script: """
                                set CX_APIKEY=%CX_APIKEY%
                                ${env.CX_CLI_PATH} results show --scan-id ${env.SCAN_ID}
                            """,
                            returnStdout: true
                        ).trim()

                        echo "Scan Results Output:\n${resultOutput}"

                        // Check for Critical or High vulnerabilities
                        if (resultOutput =~ /Critical\s*:\s*[1-9]\d*/ || resultOutput =~ /High\s*:\s*[1-9]\d*/) {
                            error("❌ High or Critical severity vulnerabilities found! Failing the pipeline.")
                        } else {
                            echo "✅ No high or critical severity vulnerabilities detected."
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline finished. Cleaning up workspace..."
            cleanWs()
        }
        failure {
            echo "Build failed!"
        }
    }
}
