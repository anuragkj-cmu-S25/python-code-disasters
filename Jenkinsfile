pipeline {
    agent any
    
    environment {
        GCP_PROJECT_ID = credentials('gcp-project-id')
        SONAR_TOKEN = credentials('sonar-token')
        SONARQUBE_URL = 'http://sonarqube-sonarqube.sonarqube.svc.cluster.local:9000'
        DATAPROC_CLUSTER = 'hadoop-cluster'
        DATAPROC_REGION = 'us-central1'
        GCS_BUCKET = "gs://${GCP_PROJECT_ID}-hadoop-output"
        SONAR_SCANNER_VERSION = '4.8.0.2856'
        SONAR_SCANNER_HOME = "${WORKSPACE}/.sonar/sonar-scanner-${SONAR_SCANNER_VERSION}-linux"
        GCLOUD_HOME = "${WORKSPACE}/.gcloud"
        PATH = "${WORKSPACE}/.gcloud/google-cloud-sdk/bin:${PATH}"
        CLOUDSDK_AUTH_CREDENTIAL_FILE_OVERRIDE = "/var/secrets/gcp/key.json"
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo '========================================='
                echo 'Stage 1: Checking out code from GitHub'
                echo '========================================='
                checkout scm
                sh 'ls -la'
                sh 'pwd'
            }
        }
        
        stage('Setup Google Cloud SDK') {
            steps {
                echo '========================================='
                echo 'Stage 2: Setting up Google Cloud SDK'
                echo '========================================='
                script {
                    sh '''
                        if [ ! -d "${GCLOUD_HOME}/google-cloud-sdk" ]; then
                            echo "Installing Google Cloud SDK..."
                            mkdir -p ${GCLOUD_HOME}
                            cd ${GCLOUD_HOME}
                            curl -sSL https://sdk.cloud.google.com | bash -s -- --disable-prompts --install-dir=${GCLOUD_HOME}
                            echo "Google Cloud SDK installed successfully"
                        else
                            echo "Google Cloud SDK already installed"
                        fi
                        
                        # Verify installation
                        gcloud version
                        
                        # Configure authentication
                        gcloud config set project ${GCP_PROJECT_ID}
                        echo "Authenticated with GCP project: ${GCP_PROJECT_ID}"
                    '''
                }
            }
        }
        
        stage('Setup SonarQube Scanner') {
            steps {
                echo '========================================='
                echo 'Stage 3: Setting up SonarQube Scanner'
                echo '========================================='
                script {
                    // Check if scanner already exists, if not download it
                    sh '''
                        if [ ! -d "${SONAR_SCANNER_HOME}" ]; then
                            echo "Downloading SonarQube Scanner..."
                            mkdir -p ${WORKSPACE}/.sonar
                            cd ${WORKSPACE}/.sonar
                            curl -sSLO https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-${SONAR_SCANNER_VERSION}-linux.zip
                            unzip -q sonar-scanner-cli-${SONAR_SCANNER_VERSION}-linux.zip
                            rm sonar-scanner-cli-${SONAR_SCANNER_VERSION}-linux.zip
                            chmod +x ${SONAR_SCANNER_HOME}/bin/sonar-scanner
                            echo "SonarQube Scanner installed successfully"
                        else
                            echo "SonarQube Scanner already installed"
                        fi
                    '''
                }
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                echo '========================================='
                echo 'Stage 4: Running SonarQube Analysis'
                echo '========================================='
                script {
                    sh """
                        ${SONAR_SCANNER_HOME}/bin/sonar-scanner \
                            -Dsonar.projectKey=python-code-disasters \
                            -Dsonar.sources=. \
                            -Dsonar.exclusions=**/.gcloud/**,**/.sonar/**,**/hadoop_input/** \
                            -Dsonar.host.url=${SONARQUBE_URL} \
                            -Dsonar.login=${SONAR_TOKEN} \
                            -Dsonar.python.version=3
                    """
                }
            }
        }
        
        stage('Quality Gate Check') {
            steps {
                echo '========================================='
                echo 'Stage 5: Checking Quality Gate Status'
                echo '========================================='
                script {
                    // Wait for SonarQube to process the analysis
                    sleep(time: 30, unit: 'SECONDS')
                    
                    // Get quality gate status
                    def qualityGate = sh(
                        script: """
                            curl -s -u ${SONAR_TOKEN}: \
                            '${SONARQUBE_URL}/api/qualitygates/project_status?projectKey=python-code-disasters' \
                            | grep -o '"status":"[^"]*"' | cut -d'"' -f4
                        """,
                        returnStdout: true
                    ).trim()
                    
                    echo "Quality Gate Status: ${qualityGate}"
                    
                    // Get blocker issues count
                    def blockerCount = sh(
                        script: """
                            curl -s -u ${SONAR_TOKEN}: \
                            '${SONARQUBE_URL}/api/issues/search?componentKeys=python-code-disasters&severities=BLOCKER&resolved=false' \
                            | grep -o '"total":[0-9]*' | head -1 | cut -d':' -f2
                        """,
                        returnStdout: true
                    ).trim()
                    
                    echo "Blocker Issues Count: ${blockerCount}"
                    
                    // Store blocker count for next stage
                    env.BLOCKER_COUNT = blockerCount
                    
                    if (blockerCount.toInteger() > 0) {
                        echo "⚠️  WARNING: Found ${blockerCount} blocker issue(s)!"
                        echo "❌ Hadoop job will NOT be executed due to blocker issues."
                        env.RUN_HADOOP = 'false'
                    } else {
                        echo "✅ SUCCESS: No blocker issues found!"
                        echo "✅ Hadoop job will be executed."
                        env.RUN_HADOOP = 'true'
                    }
                }
            }
        }
        
        stage('Upload MapReduce Job to GCS') {
            when {
                expression { env.RUN_HADOOP == 'true' }
            }
            steps {
                echo '========================================='
                echo 'Stage 6: Uploading MapReduce Job to GCS'
                echo '========================================='
                script {
                    // Create the MapReduce Python script
                    sh '''
                        cat > line_counter.py << 'EOF'
#!/usr/bin/env python3
"""
Hadoop MapReduce job to count lines in each file of the repository.
"""
import sys
import os
from collections import defaultdict

def mapper():
    """
    Mapper: Read from stdin and emit (filename, 1) for each line
    """
    current_file = os.environ.get('mapreduce_map_input_file', 'unknown')
    # Extract just the filename from the full path
    filename = current_file.split('/')[-1] if '/' in current_file else current_file
    
    for line in sys.stdin:
        # Emit filename and count of 1 for each line
        print(f"{filename}\\t1")

def reducer():
    """
    Reducer: Sum up line counts for each file
    """
    current_file = None
    current_count = 0
    
    for line in sys.stdin:
        line = line.strip()
        if not line:
            continue
            
        parts = line.split('\\t')
        if len(parts) != 2:
            continue
            
        filename, count = parts
        
        try:
            count = int(count)
        except ValueError:
            continue
        
        if current_file == filename:
            current_count += count
        else:
            if current_file is not None:
                # Output the result
                print(f'"{current_file}": {current_count} lines')
            current_file = filename
            current_count = count
    
    # Output the last file
    if current_file is not None:
        print(f'"{current_file}": {current_count} lines')

if __name__ == '__main__':
    # Determine if we're running as mapper or reducer
    if len(sys.argv) > 1 and sys.argv[1] == 'reduce':
        reducer()
    else:
        mapper()
EOF
                        chmod +x line_counter.py
                    '''
                    
                    // Upload to GCS
                    sh """
                        gcloud storage cp line_counter.py ${GCS_BUCKET}/scripts/
                        echo "✅ MapReduce job uploaded to GCS"
                    """
                }
            }
        }
        
        stage('Prepare Repository for Hadoop') {
            when {
                expression { env.RUN_HADOOP == 'true' }
            }
            steps {
                echo '========================================='
                echo 'Stage 7: Preparing Repository Files'
                echo '========================================='
                script {
                    // Create a tarball of all Python files
                    sh '''
                        mkdir -p hadoop_input
                        find . -name "*.py" -type f | while read file; do
                            cp "$file" "hadoop_input/"
                        done
                        ls -la hadoop_input/
                    '''
                    
                    // Upload input files to GCS
                    sh """
                        gcloud storage rm -r ${GCS_BUCKET}/input/ || true
                        gcloud storage cp -r hadoop_input/* ${GCS_BUCKET}/input/
                        echo "✅ Input files uploaded to ${GCS_BUCKET}/input/"
                    """
                }
            }
        }
        
        stage('Run Hadoop MapReduce Job') {
            when {
                expression { env.RUN_HADOOP == 'true' }
            }
            steps {
                echo '========================================='
                echo 'Stage 8: Running Hadoop MapReduce Job'
                echo '========================================='
                script {
                    // Clean up previous output
                    sh """
                        gcloud storage rm -r ${GCS_BUCKET}/output/ || true
                    """
                    
                    // Submit the Hadoop Streaming job
                    sh """
                        gcloud dataproc jobs submit hadoop \
                            --cluster=${DATAPROC_CLUSTER} \
                            --region=${DATAPROC_REGION} \
                            --class=org.apache.hadoop.streaming.HadoopStreaming \
                            --jars=file:///usr/lib/hadoop-mapreduce/hadoop-streaming.jar \
                            -- \
                            -input ${GCS_BUCKET}/input/* \
                            -output ${GCS_BUCKET}/output \
                            -mapper "${GCS_BUCKET}/scripts/line_counter.py" \
                            -reducer "${GCS_BUCKET}/scripts/line_counter.py reduce" \
                            -file ${GCS_BUCKET}/scripts/line_counter.py
                    """
                    
                    echo "✅ Hadoop job submitted successfully!"
                }
            }
        }
        
        stage('Display Results') {
            when {
                expression { env.RUN_HADOOP == 'true' }
            }
            steps {
                echo '========================================='
                echo 'Stage 9: Displaying Hadoop Job Results'
                echo '========================================='
                script {
                    // Wait a bit for the job to complete
                    sleep(time: 10, unit: 'SECONDS')
                    
                    // Fetch and display results
                    sh """
                        echo ""
                        echo "========================================="
                        echo "HADOOP JOB RESULTS - LINE COUNT PER FILE"
                        echo "========================================="
                        gcloud storage cat ${GCS_BUCKET}/output/part-* || echo "Results not ready yet, check GCS bucket manually"
                        echo ""
                        echo "========================================="
                        echo "Results also available at:"
                        echo "${GCS_BUCKET}/output/"
                        echo "========================================="
                    """
                }
            }
        }
    }
    
    post {
        always {
            echo '========================================='
            echo 'Pipeline Execution Complete'
            echo '========================================='
            script {
                if (env.BLOCKER_COUNT?.toInteger() > 0) {
                    echo "❌ Pipeline completed with ${env.BLOCKER_COUNT} blocker issues"
                    echo "❌ Hadoop job was NOT executed"
                } else if (env.RUN_HADOOP == 'true') {
                    echo "✅ Pipeline completed successfully"
                    echo "✅ Hadoop job executed and results available"
                    echo "📊 View results: ${GCS_BUCKET}/output/"
                } else {
                    echo "⚠️  Pipeline completed with warnings"
                }
            }
        }
        success {
            echo '✅ Build Status: SUCCESS'
        }
        failure {
            echo '❌ Build Status: FAILED'
        }
    }
}
