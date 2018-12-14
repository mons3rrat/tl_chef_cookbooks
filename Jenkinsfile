readProperties = loadConfigurationFile 'buildConfiguration'
pipeline {
  agent any
  environment {
    AWS_ACCESS_KEY_ID = credentials('aws_access_key')
    AWS_SECRET_ACCESS_KEY = credentials('aws_secret_key')
    TF_VAR_my_public_key_path = credentials('ssh-public-key')
    TF_VAR_my_private_key_path = credentials('ssh-private-key')
    TOKEN = credentials('gh-token')
    TF_PLUGIN_CACHE_DIR = '/plugins'
  }
  triggers { pollSCM('H/5 * * * *') }
  stages {
    stage('run foodcritic'){
      agent {
        docker {
          image readProperties.imageChefdk
          }
        }
      when { expression{ env.BRANCH_NAME ==~ /dev.*/ || env.BRANCH_NAME ==~ /PR.*/ || env.BRANCH_NAME ==~ /feat.*/ } }
      steps{
        echo "############ Running Foodcritic ############"
        sh 'foodcritic -B cookbook/apache/ || exit 0'
      }
    }
    stage('run rubocop'){
      agent {
        docker {
          image readProperties.imageChefdk
        }
      }
      when { expression{ env.BRANCH_NAME ==~ /dev.*/ || env.BRANCH_NAME ==~ /PR.*/ || env.BRANCH_NAME ==~ /feat.*/ } }
      steps{
        echo "############ Running Rubocop ############"
        //sh 'rubocop .cookbooks/apache/ || exit 0'
      }
    }
    stage('unit test'){
      agent {
        docker {
          image readProperties.imageChefdk
        }
      }
      when { expression{ env.BRANCH_NAME ==~ /dev.*/ || env.BRANCH_NAME ==~ /PR.*/ || env.BRANCH_NAME ==~ /feat.*/ } }
      steps{
        echo "############ Running UnitTest ############"
        sh 'chef exec rspec'
      }
    }
    stage('integration test'){
      agent {
        docker {
          image readProperties.imageChefdk
        }
      }
      when { expression{ env.BRANCH_NAME ==~ /dev.*/ || env.BRANCH_NAME ==~ /PR.*/ || env.BRANCH_NAME ==~ /feat.*/ } }
      steps{
        echo "############ Running Integration Test ############"
      }
    }
    stage("Approval step"){
      agent none
      steps{
        input message: "Do you want to create a PR to master branch?", ok: 'Approve'
      }
    }
    stage('Generate PR'){
      agent {
        docker {
          image readProperties.imagePipeline
        }
      }
      when { expression{ env.BRANCH_NAME ==~ /feat.*/ } }
      steps{
        createPR "mons3rrat", readProperties.title, "dev", env.BRANCH_NAME, "xfrarod"
        slackSend baseUrl: readProperties.slack, channel: '#cloudeng_notification', color: '#00FF00', message: "Please review and approve PR to merge changes to dev branch : https://github.com/xfrarod/tl_chef_cookbooks/pulls"
        }
    }
    stage('Knife cookbook upload'){
      agent {
        docker {
          image readProperties.imageChefdk
        }
      }
      when { expression{ env.BRANCH_NAME == "master" } }
      steps{
        echo "###knife cookbook upload####"
      }
    }
  }
  post {
    success {
      slackSend baseUrl: readProperties.slack, channel: '#cloudeng_notification', color: '#00FF00', message: "SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})"
    }
    failure {
      script{
        def commiter_user = sh "git log -1 --format='%ae'"
        slackSend baseUrl: readProperties.slack, channel: '##cloudeng_notification', color: '#FF0000', message: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})"
      }
    }
    always {
          sh "docker system prune -f"
    }
  }
}
