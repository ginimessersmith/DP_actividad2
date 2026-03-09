pipeline {
  agent any
  environment {
    IMAGE_NAME = "demo-ci-cd:latest"
    REMOTE_IP = "192.168.0.55"
    REMOTE_USER = "ginimessersmith"
    SSH_KEY = "/var/jenkins_home/.ssh/id_ed25519"
    REMOTE_DIR = "/home/ginimessersmith/artifacts/spring-boot-private"
    DEPLOY_SCRIPT = "/opt/spring-boot-app/deploy.sh"
    JAR_NAME = "demo-0.0.1-SNAPSHOT.jar"
  }
  stages {
    stage('Checkout') {
      steps {
        git branch: 'main', url: 'https://github.com/ginimessersmith/DP_springboot.git', credentialsId: '2'
        sh 'echo "checkout del repositorio completado"'
      }
    }
    
    stage('Build') {
      steps {
        sh 'mvn clean compile'
      }
    }

    stage('Unit Tests & Package') {
      steps {
        sh 'echo "Ejecutando tests y creando JAR..."'
        sh 'mvn package'
      }
    }
    
    stage('SAST - Semgrep') {
        steps {
            
            sh 'semgrep --config=auto --json --output semgrep_result.json .'
            sh 'python3 -m json.tool semgrep_result.json'
        }
    }

    stage('Transfer Artifact') {
      steps {
        sh 'echo "Enviando JAR al servidor..."'
        // Transferimos el archivo una sola vez
        sh "scp -o StrictHostKeyChecking=no -i ${SSH_KEY} target/${JAR_NAME} ${REMOTE_USER}@${REMOTE_IP}:${REMOTE_DIR}/"
      }
    }

    stage('Deploy Canary (8082)') {
      steps {
        sh 'echo "Iniciando Canary Release en puerto 8082..."'
        // AQUI ESTABA EL ERROR: Agregamos el argumento '8082' al final
        sh "ssh -i ${SSH_KEY} ${REMOTE_USER}@${REMOTE_IP} 'sudo ${DEPLOY_SCRIPT} ${REMOTE_DIR}/${JAR_NAME} 8082'"
      }
    }
    
    stage('DAST Security Scan (Vía SSH en Ubuntu)') {
      steps {
        sh 'echo "Iniciando escaneo DAST rápido con ZAP Bare..."'
        sh """
          ssh -o StrictHostKeyChecking=no -i ${SSH_KEY} ${REMOTE_USER}@${REMOTE_IP} 'docker run --rm --network host -v /home/${REMOTE_USER}:/zap/wrk/:rw -t zaproxy/zap-bare zap.sh -cmd -quickurl http://localhost:8082 -quickout /zap/wrk/zap_report.html'
        """
        
        sh 'echo "Recuperando el reporte de seguridad..."'
        sh "scp -o StrictHostKeyChecking=no -i ${SSH_KEY} ${REMOTE_USER}@${REMOTE_IP}:/home/${REMOTE_USER}/zap_report.html ."
      }
    }

    stage('Deploy Production (8081)') {
      steps {
        sh 'echo " Promoviendo a Producción en puerto 8081..."'
        // Si el paso anterior no falló, desplegamos en 8081
        sh "ssh -i ${SSH_KEY} ${REMOTE_USER}@${REMOTE_IP} 'sudo ${DEPLOY_SCRIPT} ${REMOTE_DIR}/${JAR_NAME} 8081'"
      }
    }
  }
  
  post {
    always {
      archiveArtifacts artifacts: 'target/*.jar, zap_report.html, semgrep_result.json', fingerprint: true, allowEmptyArchive: true
    }
    success {
      echo 'spliegue completado exitosamente en ambos nodos.'
    }
    failure {
      echo 'El despliegue falló. Revisa los logs.'
    }
  }
}