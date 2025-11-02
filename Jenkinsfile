pipeline {
  agent { label 'build' }

  environment {
    registry           = "zoum444/democicd"
    registryCredential = "dockerhub"
    JAVA8              = "/usr/lib/jvm/java-8-openjdk-amd64"
    JAVA17             = "/usr/lib/jvm/java-17-openjdk-amd64"
    DC_DATA_DIR        = "${WORKSPACE}/.dependency-check"
  }

  parameters {
    password(name: 'PASSWD', defaultValue: '', description: 'Please Enter your Gitlab password')
  }

  stages {

    stage('Stage I: Build & Test (JDK8)') {
      steps {
        git branch: 'main', credentialsId: 'GithubCred', url: 'https://github.com/zoumana001/spring-build-pipeline.git'
        echo "Building Jar Component ..."
        sh '''
          export JAVA_HOME="$JAVA8"; export PATH="$JAVA_HOME/bin:$PATH"
          java -version
          mvn -v
          mvn -B clean test
        '''
      }
    }

    stage('Stage II: Code Coverage (JDK8)') {
      steps {
        echo "Running Code Coverage ..."
        sh '''
          export JAVA_HOME="$JAVA8"; export PATH="$JAVA_HOME/bin:$PATH"
          mvn -B -Ddependency-check.skip=true verify
        '''
      }
      post {
        always {
          // If HTML Publisher plugin is installed, uncomment block below and remove archiveArtifacts
          // publishHTML(target: [
          //   reportDir: 'target/site/jacoco',
          //   reportFiles: 'index.html',
          //   reportName: 'JaCoCo Coverage'
          // ])
          archiveArtifacts artifacts: 'target/site/jacoco/**', fingerprint: true
        }
      }
    }

    stage('Stage III: SCA (OWASP, JDK17)') {
      steps {
        echo "Running Software Composition Analysis using OWASP Dependency-Check ..."
        withCredentials([string(credentialsId: 'NVD_API_KEY', variable: 'NVD_API_KEY')]) {
          sh '''
            set -e
            export JAVA_HOME="$JAVA17"; export PATH="$JAVA_HOME/bin:$PATH"

            # 1) Verify the key exists (don’t print it)
            if [ -z "$NVD_API_KEY" ]; then
              echo "ERROR: NVD_API_KEY env var not set (check Jenkins credentials ID)."
              exit 9
            else
              echo "NVD_API_KEY is present (masked)."
            fi

            # 2) Trim hidden newlines/carriage-returns that break auth
            NVD_API_KEY_CLEAN="$(printf "%s" "$NVD_API_KEY" | tr -d "\\r\\n")"

            # 3) Quick connectivity test to NVD API (should be 200)
            CODE=$(curl -s -o /dev/null -w "%{http_code}" \
              -H "apiKey: $NVD_API_KEY_CLEAN" \
              "https://services.nvd.nist.gov/rest/json/cves/2.0?resultsPerPage=1")
            echo "NVD connectivity HTTP code: $CODE"
            if [ "$CODE" != "200" ]; then
              echo "ERROR: NVD API not reachable or key rejected (HTTP $CODE)."
              echo "If you use a proxy, add -Dhttps.proxyHost/Port to the Maven command below."
              exit 10
            fi

            # 4) Run Dependency-Check (cache DB in workspace to avoid constant downloads)
            mkdir -p "$DC_DATA_DIR"
            mvn -B \
              -DskipTests \
              -DdataDirectory="$DC_DATA_DIR" \
              -Dnvd.api.key="$NVD_API_KEY_CLEAN" \
              -Dformat=HTML \
              -DoutputDirectory=target/dependency-check \
              org.owasp:dependency-check-maven:check
          '''
        }
      }
      post {
        always {
          archiveArtifacts artifacts: 'target/dependency-check/*', fingerprint: true
          // If HTML Publisher is installed, you can publish the HTML too:
          // publishHTML(target: [
          //   reportDir: 'target/dependency-check',
          //   reportFiles: 'dependency-check-report.html',
          //   reportName: 'Dependency-Check Report'
          // ])
        }
      }
    }

    stage('Stage IV: SAST (SonarQube, JDK8)') {
      steps {
        echo "Running Static application security testing using SonarQube Scanner ..."
        withSonarQubeEnv('mysonarqube') {
          sh '''
            export JAVA_HOME="$JAVA8"; export PATH="$JAVA_HOME/bin:$PATH"
            mvn -B sonar:sonar \
              -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml \
              -Dsonar.dependencyCheck.jsonReportPath=target/dependency-check/dependency-check-report.json \
              -Dsonar.dependencyCheck.htmlReportPath=target/dependency-check/dependency-check-report.html \
              -Dsonar.projectName=wezvatech
          '''
        }
      }
    }

    stage('Stage V: Quality Gates') {
      steps {
        echo "Running Quality Gates to verify the code quality"
        script {
          timeout(time: 1, unit: 'MINUTES') {
            def qg = waitForQualityGate()
            if (qg.status != 'OK') {
              error "Pipeline aborted due to quality gate failure: ${qg.status}"
            }
          }
        }
      }
    }

    stage('Stage VI: Build Image') {
      steps {
        echo "Build Docker Image"
        script {
          docker.withRegistry('', registryCredential) {
            def myImage = docker.build("${registry}:${BUILD_NUMBER}")
            myImage.push()
          }
        }
      }
    }

    stage('Stage VII: Scan Image') {
      steps {
        echo "Scanning Image for Vulnerabilities"
        sh '''
          trivy image --scanners vuln --offline-scan ${registry}:${BUILD_NUMBER} > trivyresults.txt || true
        '''
        archiveArtifacts artifacts: 'trivyresults.txt', fingerprint: true
      }
    }

    stage('Stage VIII: Smoke Test') {
      steps {
        echo "Smoke Test the Image"
        sh """
          docker run -d --name smokerun -p 8080:8080 ${registry}:${BUILD_NUMBER}
          sleep 90; ./check.sh
          docker rm --force smokerun
        """
      }
    }

    stage('Stage IX: Trigger Deployment') {
      steps {
        script {
          def TAG = "${BUILD_NUMBER}"
          echo "Trigger CD Pipeline"
          build wait: false, job: 'springboot-cd-pipeline', parameters: [
            password(name: 'PASSWD', description: 'Please Enter your Gitlab password', value: params.PASSWD),
            string(name: 'IMAGETAG', value: TAG)
          ]
        }
      }
    }

  } // end stages
} // end pipeline
