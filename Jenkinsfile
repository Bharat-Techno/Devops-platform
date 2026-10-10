pipeline {
  agent {
    kubernetes {
      inheritFrom 'jenkins-nexus'
      defaultContainer 'jnlp'
    }
  }

  environment {
    IMAGE = "192.168.56.11:8082/devops-platform"
    VERSION = "v${BUILD_NUMBER}"
    NAMESPACE = "project1"
    PROM_URL = "http://monitoring-kube-prometheus-prometheus.monitoring.svc:9090"
    ANSIBLE_HOST_KEY_CHECKING = "False"
  }

  stages {

    stage('Init Version') {
      steps {
        script {
          env.APP_VERSION = "2.0.${BUILD_NUMBER}-${System.currentTimeMillis()}"
        }
      }
    }

    stage('Clone Code') {
      steps {
        git url: 'git@github.com:BharatDasa/devops-platform.git',
            credentialsId: 'github-ssh'
      }
    }

    stage('Build & Upload Artifact') {
      steps {
        container('maven') {
          sh '''
          mvn clean deploy -U \
            -DskipTests \
            -Drevision=${APP_VERSION} \
            -Dchangelist=""
          '''
        }
      }
    }

    stage('Build & Push Image') {
      steps {
        container('kaniko') {
          sh '''
          /kaniko/executor \
            --dockerfile=Dockerfile \
            --context="$(pwd)" \
            --destination=${IMAGE}:latest \
            --destination=${IMAGE}:${VERSION} \
            --insecure \
            --skip-tls-verify
          '''
        }
      }
    }

    stage('Ansible Auto-Fix') {
      steps {
        container('ansible') {
          withCredentials([
            sshUserPrivateKey(
              credentialsId: 'ansible-ssh-key',
              keyFileVariable: 'SSH_KEY'
            )
          ]) {
            sh '''
            chmod 600 $SSH_KEY
            ansible-playbook -i ansible/inventory.ini ansible/auto-fix.yml \
              --private-key=$SSH_KEY -u bharat
            '''
          }
        }
      }
    }

    // 🔥 PARALLEL: TRIVY + ANSIBLE VALIDATION
    stage('Validation + Security') {
      parallel {

        // ✅ ANSIBLE VALIDATION
        stage('Ansible Validation') {
          steps {
            container('ansible') {
              withCredentials([
                sshUserPrivateKey(
                  credentialsId: 'ansible-ssh-key',
                  keyFileVariable: 'SSH_KEY'
                )
              ]) {
                sh '''
                chmod 600 $SSH_KEY
                ansible-playbook -i ansible/inventory.ini ansible/precheck.yml \
                  --private-key=$SSH_KEY -u bharat
                '''
              }
            }
          }
        }

        // 🔐 TRIVY HTML REPORT (SAFE FIRST RUN)
        stage('Trivy HTML Report') {
          steps {
            container('trivy') {
              sh '''
              rm -rf /tmp/trivy-cache-html
              mkdir -p /tmp/trivy-cache-html
              mkdir -p reports

              wget -q https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/html.tpl -O html.tpl

              trivy image \
                --cache-dir /tmp/trivy-cache-html \
                --timeout 45m \
                --scanners vuln \
                --format template \
                --template "@html.tpl" \
                -o reports/trivy-report.html \
                ${IMAGE}:${VERSION} || true
              '''
            }
          }
        }

        // 🔐 TRIVY CRITICAL GATE
        stage('Trivy Critical Scan') {
          steps {
            container('trivy') {
              sh '''
              rm -rf /tmp/trivy-cache-gate
              mkdir -p /tmp/trivy-cache-gate

              trivy image \
                --cache-dir /tmp/trivy-cache-gate \
                --timeout 45m \
                --scanners vuln \
                --severity CRITICAL \
                --ignore-unfixed \
                --exit-code 1 \
                ${IMAGE}:${VERSION}
              '''
            }
          }
        }

      }
    }

    // 📊 REPORT IN JENKINS UI
    stage('Publish Trivy Report') {
      steps {
        publishHTML([
          reportDir: 'reports',
          reportFiles: 'trivy-report.html',
          reportName: 'Trivy Security Report',
          keepAll: true,
          alwaysLinkToLastBuild: true,
          allowMissing: true
        ])
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        sh '''
        echo "===== DOWNLOAD kubectl ====="
        curl -LO https://dl.k8s.io/release/v1.29.0/bin/linux/amd64/kubectl
        chmod +x kubectl

        echo "===== CREATE NAMESPACE ====="
        ./kubectl create namespace ${NAMESPACE} || true

        echo "===== SET IMAGE ====="
        sed -i "s|IMAGE_PLACEHOLDER|${IMAGE}:${VERSION}|g" k8s/deployment.yaml

        echo "===== APPLY MANIFESTS ====="
        ./kubectl apply -f k8s/

        echo "===== APPLY SERVICEMONITOR ====="
        ./kubectl apply -f k8s/servicemonitor.yaml

        echo "===== ROLLOUT STATUS ====="
        ./kubectl rollout status deployment/hello-app -n ${NAMESPACE} --timeout=180s
        '''
      }
    }

    stage('Deploy HPA') {
      steps {
        sh '''
        ./kubectl apply -f k8s/hpa.yaml
        ./kubectl get hpa -n ${NAMESPACE}
        '''
      }
    }

    stage('Verify Monitoring') {
      steps {
        sh '''
        RESPONSE=$(curl -s ${PROM_URL}/api/v1/query?query=up)

        if echo "$RESPONSE" | grep -q '"status":"success"'; then
          echo "✅ Prometheus reachable"
        else
          echo "❌ Prometheus failed"
          exit 1
        fi
        '''
      }
    }

    stage('Health Check & Rollback') {
      steps {
        sh '''
        READY=$(./kubectl get deployment hello-app -n ${NAMESPACE} \
          -o jsonpath='{.status.readyReplicas}')

        DESIRED=$(./kubectl get deployment hello-app -n ${NAMESPACE} \
          -o jsonpath='{.spec.replicas}')

        if [ "$READY" != "$DESIRED" ]; then
          echo "❌ Rolling back..."
          ./kubectl rollout undo deployment/hello-app -n ${NAMESPACE}
          exit 1
        fi

        echo "===== HPA STATUS ====="
        ./kubectl get hpa -n ${NAMESPACE}

        echo "✅ Deployment Healthy"
        '''
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'reports/*.html', allowEmptyArchive: true
    }

    success {
      echo "✅ SUCCESS: ${IMAGE}:${VERSION} deployed to ${NAMESPACE}"
      echo "SUCCESS: Full DevSecOps pipeline completed"
      echo "🌐 URL: http://hello.192.168.56.100.nip.io"
    }

    failure {
      echo "❌ FAILED: Blocked by security or rolled back"
    }
  }
}
