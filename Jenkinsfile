pipeline {
  agent none
  stages {
    stage('worker-build') {
      agent {
        docker {
          image 'maven:3.8.5-jdk-11-slim'
          args '-v $HOME/.m2:/root/.m2'
        }

      }
      when {
        changeset '**/worker/**'
      }
      steps {
        echo 'Compiling worker app..'
        dir(path: 'worker') {
          sh 'mvn compile'
        }

      }
    }

    stage('worker test') {
      agent {
        docker {
          image 'maven:3.8.5-jdk-11-slim'
          args '-v $HOME/.m2:/root/.m2'
        }

      }
      when {
        changeset '**/worker/**'
      }
      steps {
        echo 'Running Unit Tets on worker app.'
        dir(path: 'worker') {
          sh 'mvn clean test'
        }

      }
    }

    stage('worker-package') {
      agent {
        docker {
          image 'maven:3.8.5-jdk-11-slim'
          args '-v $HOME/.m2:/root/.m2'
        }

      }
      when {
        branch 'master'
        changeset '**/worker/**'
      }
      steps {
        echo 'Packaging worker app'
        dir(path: 'worker') {
          sh 'mvn package -DskipTests'
          archiveArtifacts(artifacts: '**/target/*.jar', fingerprint: true)
        }

      }
    }

    stage('worker-docker-package') {
      agent any
      when {
        changeset '**/worker/**'
        branch 'master'
      }
      steps {
        echo 'Packaging worker app with docker'
        script {
          docker.withRegistry('https://index.docker.io/v1/', 'dockerlogin') {
            def workerImage = docker.build("okapetanios/worker:v${env.BUILD_ID}", './worker')
            workerImage.push()
            workerImage.push("${env.BRANCH_NAME}")
            workerImage.push('latest')
          }
        }

      }
    }

    stage('result-build') {
      agent {
        docker {
          image 'node:8.16.0-alpine'
        }

      }
      when {
        changeset '**/result/**'
      }
      steps {
        echo 'Compiling result app..'
        dir(path: 'result') {
          sh 'npm install'
        }

      }
    }

    stage('result-test') {
      agent {
        docker {
          image 'node:8.16.0-alpine'
        }

      }
      when {
        changeset '**/result/**'
      }
      steps {
        echo 'Running Unit Tests on result app..'
        dir(path: 'result') {
          sh 'npm install'
          sh 'npm test'
        }

      }
    }

    stage('result-docker-package') {
      agent any
      when {
        changeset '**/result/**'
        branch 'master'
      }
      steps {
        echo 'Packaging result app with docker'
        script {
          docker.withRegistry('https://index.docker.io/v1/', 'dockerlogin') {
            def resultImage = docker.build("okapetanios/result:v${env.BUILD_ID}", './result')
            resultImage.push()
            resultImage.push("${env.BRANCH_NAME}")
            resultImage.push('latest')
          }
        }

      }
    }

    stage('vote-build') {
      agent {
        docker {
          image 'python:2.7.16-slim'
          args '--user root'
        }

      }
      when {
        changeset '**/vote/**'
      }
      steps {
        echo 'Compiling vote app.'
        dir(path: 'vote') {
          sh 'pip install -r requirements.txt'
        }

      }
    }

    stage('vote-test') {
      agent {
        docker {
          image 'python:2.7.16-slim'
          args '--user root'
        }

      }
      when {
        changeset '**/vote/**'
      }
      steps {
        echo 'Running Unit Tests on vote app.'
        dir(path: 'vote') {
          sh 'pip install -r requirements.txt'
          sh 'nosetests -v'
        }

      }
    }

    stage('vote integration'){ 
    agent any 
    when{ 
      changeset "**/vote/**" 
      branch 'master' 
    } 
    steps{ 
      echo 'Running Integration Tests on vote app' 
      dir('vote'){ 
        sh 'sh integration_test.sh' 
      } 
    } 
} 


    stage('vote-docker-package') {
      agent any
      steps {
        echo 'Packaging vote app with docker'
        script {
          docker.withRegistry('https://index.docker.io/v1/', 'dockerlogin') {
            // ./vote is the path to the Dockerfile that Jenkins will find from the Github repo
            def voteImage = docker.build("okapetanios/vote:v${env.GIT_COMMIT}", "./vote")
            voteImage.push()
            voteImage.push("${env.BRANCH_NAME}")
            voteImage.push("latest")
          }
        }

      }
    }

      stage('Argo CD') {
      agent {
        docker{
          image 'argoproj/argocd:v2.4.5'
        }
      }
     
      environment {
        GIT_CREDS = credentials('eeganlf-github')
        HELM_GIT_REPO_URL = "github.com/eeganlf/vote-deploy.git"
        GIT_REPO_EMAIL = 'eegan@linuxfoundaton.org'
        GIT_REPO_BRANCH = "master"
          
       // Update above variables with your user details
      }
      steps {
        echo 'Updating GitOps Repository'
         //script {
         sh "git clone https://${env.HELM_GIT_REPO_URL}"
            sh "sudo git config --global user.email ${env.GIT_REPO_EMAIL}"
             // install yq
            sh "wget https://github.com/mikefarah/yq/releases/download/v4.9.6/yq_linux_amd64.tar.gz"
            sh "tar xvf yq_linux_amd64.tar.gz"
            sh "mv yq_linux_amd64 /usr/bin/yq"
            sh "git checkout -b master"
          dir("vote-deploy") {
              sh "git checkout ${env.GIT_REPO_BRANCH}"
            //install done
            sh '''#!/bin/bash
              echo $GIT_REPO_EMAIL
              echo $GIT_COMMIT
              ls -lth
              yq eval '.spec.template.spec.image = "docker.io/okapetanios/vote:" + env(GIT_COMMIT)' -i vote-ui-deployment.yaml
              cat vote-ui-deployment.yaml
              pwd
              git add vote-ui-deployment.yaml
              git commit -m 'Triggered Build'
              git push https://$GIT_CREDS_USR:$GIT_CREDS_PSW@github.com/$GIT_CREDS_USR/vote-deploy.git
            '''
           }
        //}

      }
    }

    stage('Sonarqube') {
      agent any
      when {
        branch 'master'
      }
       tools {
        jdk "JDK11" // the name you have given the JDK installation in Global Tool Configuration
      }
      environment {
        sonarpath = tool 'SonarScanner'
      }
      steps {
        echo 'Running Sonarqube Analysis..'
        // TODO: ?this must match sonar server
        withSonarQubeEnv('sonar-example-voting-app') {
          sh "${sonarpath}/bin/sonar-scanner -Dproject.settings=sonar-project.properties -Dorg.jenkinsci.plugins.durabletask.BourneShellScript.HEARTBEAT_CHECK_INTERVAL=86400"
        }

      }
    }

    stage('Quality Gate') {
      steps {
        timeout(time: 1, unit: 'HOURS') {
          waitForQualityGate true
        }

      }
    }

    stage('deploy to dev') {
      agent any
      when {
        branch 'master'
      }
      steps {
        echo 'Deploy instavote app with docker compose'
        sh 'docker-compose up -d'
      }
    }

  
  
  
  }
  post {
    always {
      echo 'Building mono pipeline for voting app is completed.'
    }

  }
}
