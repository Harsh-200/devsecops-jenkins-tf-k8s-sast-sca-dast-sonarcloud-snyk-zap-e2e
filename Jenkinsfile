pipeline {
  agent any
  tools { 
        maven 'Maven_3_2_5'  
    }
   stages{
    stage('CompileandRunSonarAnalysis') {
            steps {	
		sh 'mvn clean verify sonar:sonar -Dsonar.projectKey=sampleapp-devsecops_sampleproject -Dsonar.organization=sampleapp-devsecops -Dsonar.host.url=https://sonarcloud.io -Dsonar.token=e80fbad06e6e837fb08d2e9fda2bae5081fe580c'
			}
    }

	stage('RunSCAAnalysisUsingSnyk') {
            steps {		
				withCredentials([string(credentialsId: 'SNYK_TOKEN', variable: 'SNYK_TOKEN')]) {
					sh 'mvn snyk:test -fn'
				}
			}
    }	
	   
	stage('Build') { 
            steps { 
               withDockerRegistry([credentialsId: "dockerlogin", url: ""]) {
                 script{
                 app =  docker.build("myproject")
                 }
               }
            }
    }

	stage('Push') {
            steps {
                script{
                    docker.withRegistry('https://692859936321.dkr.ecr.us-east-2.amazonaws.com', 'ecr:us-east-2:aws-credentials') {
                    app.push("latest")
                    }
                }
            }
    	}
	   
	stage('Kubernetes Deployment of EasyBugg Web Application') {
	   steps {
	      withKubeConfig([credentialsId: 'kubelogin']) {
		  sh('kubectl delete all --all -n devsecops')
		  sh ('kubectl apply -f deployment.yaml --namespace=devsecops')
		}
	      }
   	}
	   
	stage ('wait_for_testing'){
	   steps {
		   sh 'pwd; sleep 180; echo "Application has been deployed on K8S"'
	   	}
	   }
	   
	 

        stage('RunDASTUsingZAP') {
            steps {
                withKubeConfig([credentialsId: 'kubelogin']) {
                    script {
                        // Run ZAP and generate the report
                        sh 'zap.sh -cmd -quickurl http://$(kubectl get services/easybuggy --namespace=devsecops -o json | jq -r ".status.loadBalancer.ingress[] | .hostname") -quickprogress -quickout ${WORKSPACE}/zap_report.html'

                        // Archive the report
                        archiveArtifacts artifacts: 'zap_report.html'

                        // Parse ZAP Report and create JIRA issues
                        def zapReport = readFile("${WORKSPACE}/zap_report.html") // Read the ZAP report
                        def vulnerabilities = parseZapReport(zapReport) // Parse vulnerabilities (method defined below)

                        // Create JIRA issues for each vulnerability
                        def jiraServer = 'JIRA_SERVER' // JIRA server configuration in Jenkins
                        for (vuln in vulnerabilities) {
                            def testIssue = [
                                fields: [
                                    project: [key :'JPP'], 
                                    summary: "Vulnerability Found: ${vuln.name}",
                                    description: """
                                        Severity: ${vuln.severity}
                                        URL: ${vuln.url}
                                        Description: ${vuln.description}
                                    """,
                                    issuetype: [name: 'Bug'], 
                                    assignee: [username: 'harshini prakash']
                                ]
                            ]
                            def response = jiraNewIssue(issue: testIssue, site: jiraServer)
                            echo "Created JIRA Ticket: ${response.data.key}"
                        }
                    }
                }
            }
        }
  }
}

// Helper function to parse ZAP report and extract vulnerabilities
def parseZapReport(reportContent) {
    // Placeholder function: Parse ZAP report and return a list of vulnerabilities
    // Each vulnerability contains 'name', 'severity', 'url', and 'description'.
    // Actual implementation depends on the ZAP report format (e.g., XML or HTML parsing).
    def vulnerabilities = []
    // Example vulnerability for demonstration
    vulnerabilities << [
        name: "SQL Injection",
        severity: "High",
        url: "http://example.com/vulnerable-endpoint",
        description: "The application is vulnerable to SQL injection."
    ]
    return vulnerabilities
}
