pipeline {
    agent any

    tools {
        allure 'Default-Allure'
    }

    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['QA','BETA'],
            description: 'Select environment to run tests'
        )
    }

    environment {
        PYTHON_VERSION       = '3.9.18'
        ALLURE_RESULTS       = 'allure-results'
        ALLURE_REPORT_FOLDER = 'allure-report'
        ALLURE_COMBINED_DIR  = 'combined-report-output' // Directory for allure-combine output
        FINAL_REPORT_NAME    = 'allure-test-report.html' // The final, user-friendly filename
        TEST_ENV             = "${params.ENVIRONMENT}"
    }

    stages {

        stage('Print Build Parameters') {
            steps {
                echo "▶ Selected environment: ${params.ENVIRONMENT}"
            }
        }

        stage('Setup Python & Dependencies') {
            steps {
                sh '''
                    #!/bin/bash
                    set -e
                    echo "--- Setting up pyenv and Python ---"
                    export PYENV_ROOT="$HOME/.pyenv"
                    export PATH="$PYENV_ROOT/bin:$PATH"
                    eval "$(pyenv init -)"

                    if ! pyenv versions --bare | grep -q "^${PYTHON_VERSION}$"; then
                      pyenv install "$PYTHON_VERSION"
                    fi
                    pyenv shell "$PYTHON_VERSION"

                    echo "--- Using Python version: $(python --version) ---"
                    echo "--- Creating virtual environment ---"
                    python -m venv venv
                    . venv/bin/activate

                    echo "--- Installing requirements ---"
                    pip install --upgrade pip
                    pip install -r requirements.txt
                '''
            }
        }

        stage('Run Pytest with Allure') {
            steps {
                sh '''
                    set -e
                    . venv/bin/activate
                    echo "--- Running tests in $TEST_ENV environment ---"
                    pytest --env="$TEST_ENV" --alluredir="$ALLURE_RESULTS"
                '''
            }
        }

        stage('Generate & Publish Allure Report') {
            steps {
                script {
                    sh '''
                        set -e
                        echo "--- Generating Allure Report folder ---"
                        export PATH="$ALLURE_HOME/bin:$PATH"
                        allure generate "$ALLURE_RESULTS" --clean -o "$ALLURE_REPORT_FOLDER"
                    '''
                    allure(
                        report: "${ALLURE_REPORT_FOLDER}",
                        results: [[path: "${ALLURE_RESULTS}"]]
                    )
                }
            }
        }

        stage('Create, Archive & Email Single-File Report') {
            steps {
                script {
                    echo 'Combining report into a single HTML file...'
                    sh """
                        set -e
                        . venv/bin/activate
                        
                        # Create the combined report in a dedicated directory
                        allure-combine "$ALLURE_REPORT_FOLDER" --dest "$ALLURE_COMBINED_DIR" --auto-create-folders
                        
                        # Move and rename the file to the workspace root with a clean name
                        mv "$ALLURE_COMBINED_DIR/complete.html" "${FINAL_REPORT_NAME}"
                        
                        echo "--- Verifying final report ---"
                        ls -la "${FINAL_REPORT_NAME}"
                        echo "File size: \$(du -h ${FINAL_REPORT_NAME} | cut -f1)"
                    """

                    // Archive the final report so it's available on the Jenkins build page
                    archiveArtifacts artifacts: "${FINAL_REPORT_NAME}"
                    
                    // Send the email with the file as a direct attachment
                    emailext(
                        from: "sathvik.bhat@lightmetrics.co",
                        to: "sathvik.bhat@lightmetrics.co, vidya.hampiholi@lightmetrics.co, divya.gajanana@lightmetrics.co", // Added all recipients
                        subject: "[${params.ENVIRONMENT}] Test Report: ${env.JOB_NAME} #${env.BUILD_NUMBER} – SUCCESS",
                        body: """
                            Hi Team,<br><br>
                            The latest automation run for <b>${env.JOB_NAME}</b> has completed successfully.<br>
                            <b>Status:</b> SUCCESS<br>
                            <b>Environment:</b> ${params.ENVIRONMENT}<br><br>
                            
                            <b>📊 Test Report:</b><br>
                            The detailed test report is attached to this email as <b>"${FINAL_REPORT_NAME}"</b>.<br><br>
                            
                            <b>📝 How to view the attachment:</b><br>
                            1. Download the attached HTML file.<br>
                            2. Double-click the file to open it directly in your browser.<br>
                            3. The report works completely offline - no server or Jenkins access is needed.<br><br>
                            
                            <b>For online viewing on Jenkins:</b> <a href="${env.BUILD_URL}allure">Click here</a><br><br>

                            Regards,<br>QA Automation Team
                        """,
                        mimeType: 'text/html',
                        attachmentsPattern: "${FINAL_REPORT_NAME}"
                    )
                }
            }
        }
    }

    post {
        failure {
            emailext(
                from: "sathvik.bhat@lightmetrics.co",
                to: "sathvik.bhat@lightmetrics.co, vidya.hampiholi@lightmetrics.co, divya.gajanana@lightmetrics.co", // Added all recipients
                subject: "[${params.ENVIRONMENT}] FAILURE: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                    Hi Team,<br><br>
                    The automation run for <b>${env.JOB_NAME}</b> has failed.<br>
                    <b>Status:</b> ${currentBuild.currentResult}<br>
                    <b>Environment:</b> ${params.ENVIRONMENT}<br>
                    <b>Build URL:</b> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a><br><br>
                    Please check the console output for details.<br><br>
                    Regards,<br>QA Automation Team
                """,
                mimeType: 'text/html'
            )
        }
        always {
            echo 'Final cleanup...'
            deleteDir()
        }
    }
}
