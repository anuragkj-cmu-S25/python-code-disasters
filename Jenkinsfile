pipeline {
    agent any
    
    environment {
        GITHUB_REPO = 'https://github.com/anuragkj-cmu-S25/python-code-disasters'
        SONAR_PROJECT_KEY = 'python-code-disasters'
        GCP_PROJECT = credentials('gcp-project-id')
        DATAPROC_CLUSTER = 'hadoop-cluster'
        DATAPROC_REGION = 'us-central1'
        DATAPROC_ZONE = 'us-central1-a'
        OUTPUT_BUCKET = "${GCP_PROJECT}-hadoop-output"
        RUN_HADOOP = 'false'
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', 
                    url: "${GITHUB_REPO}",
                    credentialsId: 'github-credentials'
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarQubeScanner'
                    withSonarQubeEnv('SonarQube') {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                                -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                                -Dsonar.sources=. \
                                -Dsonar.python.version=3
                        """
                    }
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    script {
                        def qg = waitForQualityGate()
                        echo "Quality Gate Status: ${qg.status}"
                    }
                }
            }
        }
        
        stage('Check for Blockers') {
            steps {
                script {
                    // Query SonarQube for blocker issues
                    def sonarUrl = "http://sonarqube-sonarqube.sonarqube.svc.cluster.local:9000"
                    def token = credentials('sonar-token')
                    
                    def response = sh(
                        script: """
                            curl -s -u ${token}: \
                            "${sonarUrl}/api/issues/search?componentKeys=${SONAR_PROJECT_KEY}&severities=BLOCKER&resolved=false"
                        """,
                        returnStdout: true
                    ).trim()
                    
                    // Parse blocker count
                    def blockerCount = sh(
                        script: "echo '${response}' | grep -o '\"total\":[0-9]*' | head -1 | cut -d':' -f2 || echo '0'",
                        returnStdout: true
                    ).trim().toInteger()
                    
                    echo "======================================"
                    echo "BLOCKER ISSUES FOUND: ${blockerCount}"
                    echo "======================================"
                    
                    if (blockerCount > 0) {
                        echo "❌ BLOCKERS DETECTED! Skipping Hadoop job."
                        env.RUN_HADOOP = 'false'
                    } else {
                        echo "✅ NO BLOCKERS! Proceeding with Hadoop job."
                        env.RUN_HADOOP = 'true'
                    }
                }
            }
        }
        
        stage('Run Hadoop Job') {
            when {
                expression { env.RUN_HADOOP == 'true' }
            }
            steps {
                script {
                    echo "======================================"
                    echo "Submitting job to Hadoop cluster..."
                    echo "======================================"
                    
                    // SSH to Dataproc master and run line counting
                    sh """
                        gcloud compute ssh hadoop-cluster-m \
                            --zone=${DATAPROC_ZONE} \
                            --project=${GCP_PROJECT} \
                            --command="
                                # Clone repo
                                cd /tmp
                                rm -rf python-code-disasters
                                git clone ${GITHUB_REPO}
                                cd python-code-disasters
                                
                                # Count lines
                                OUTPUT_FILE=/tmp/line_counts_\$(date +%Y%m%d_%H%M%S).txt
                                echo 'File Line Counts:' > \$OUTPUT_FILE
                                echo '==================' >> \$OUTPUT_FILE
                                
                                find . -name '*.py' -type f | while read file; do
                                    filename=\$(basename \"\$file\")
                                    linecount=\$(wc -l < \"\$file\")
                                    echo '\"\$filename\": '\$linecount >> \$OUTPUT_FILE
                                done
                                
                                # Upload to GCS
                                gsutil cp \$OUTPUT_FILE gs://${OUTPUT_BUCKET}/
                                
                                # Display results
                                echo ''
                                echo 'Results uploaded to: gs://${OUTPUT_BUCKET}/'\$(basename \$OUTPUT_FILE)
                                echo ''
                                cat \$OUTPUT_FILE
                            "
                    """
                }
            }
        }
        
        stage('Display Results') {
            when {
                expression { env.RUN_HADOOP == 'true' }
            }
            steps {
                script {
                    echo "======================================"
                    echo "HADOOP JOB RESULTS"
                    echo "======================================"
                    
                    // Get latest results from GCS
                    sh """
                        LATEST_FILE=\$(gsutil ls gs://${OUTPUT_BUCKET}/line_counts_* | tail -1)
                        echo "Latest results file: \$LATEST_FILE"
                        echo ""
                        gsutil cat \$LATEST_FILE
                    """
                    
                    echo "======================================"
                    echo "Results available in GCS bucket: ${OUTPUT_BUCKET}"
                    echo "======================================"
                }
            }
        }
    }
    
    post {
        success {
            echo "Pipeline completed successfully!"
        }
        failure {
            echo "Pipeline failed!"
        }
        always {
            cleanWs()
        }
    }
}