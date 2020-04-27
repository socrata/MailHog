@Library('socrata-pipeline-library') _

def dockerize = new com.socrata.Dockerize(steps, 'mailhog', BUILD_NUMBER)

pipeline {
  options {
    timeout(time: 60, unit: 'MINUTES')
    ansiColor('xterm')
  }

  parameters {
    string(name: 'AGENT', defaultValue: 'build-worker', description: 'Which build agent to use?')
  }

  agent {
    label params.AGENT
  }

  triggers {
    issueCommentTrigger('^retest$')
  }

  environment {
    SERVICE = 'mailhog'
    RUNNING_IN_CI = 'true'
    GITHUB_CREDENTIALS_ID = 'a3959698-3d22-43b9-95b1-1957f93e5a11'
    REPOSITORY_NAME = "${env.GIT_URL.tokenize('/')[3].split('\\.')[0]}"
    REPOSITORY_OWNER = "${env.GIT_URL.tokenize('/')[2]}"
    SLACK_CHANNEL = '#overwatch-notifications'
    SERVICE_VERSION = '1.0.0'
    ARTIFACTORY_CREDS = credentials('shared-eng-artifactory-creds')
  }

  stages {
    stage('Initialize Job Details') {
      steps {
        script {
          currentBuild.description = "${env.SERVICE} : ${env.GIT_COMMIT[0..5]} : ${env.NODE_NAME}"
        }
      }
    }

    stage('Pull Request') {
      when { changeRequest() }
      stages {
      }
    }

    stage('Master') {
      when { branch 'master' }
      stages {
        stage('Build and push image') {
          steps {
            sh 'docker build --build-arg ARTIFACTORY_USER=${ARTIFACTORY_CREDS_USR} --build-arg ARTIFACTORY_PASSWORD=${ARTIFACTORY_CREDS_PSW} -t mailhog:latest .'
            script {
              env.DOCKER_TAG = dockerize.push_tagged_image_to_all_repos('mailhog:latest', env.SERVICE_VERSION, env.GIT_COMMIT)
            }
	        }
          post {
            failure {
              slackSend(channel: "${env.SLACK_CHANNEL}", color: 'RED', message: "${env.SERVICE}: Building and pushing docker image has failed - ${env.BUILD_URL}")
            }
          }
        }
        stage('Deploy to staging') {
          steps {
            marathonDeploy()
          }
          post {
            success {
              slackSend(channel: "${env.SLACK_CHANNEL}", color: 'GREEN', message: "${env.SERVICE} (${env.DOCKER_TAG}): Successfully deployed to staging marathon")
            }
            failure {
              slackSend(channel: "${env.SLACK_CHANNEL}", color: 'RED', message: "${env.SERVICE} (${env.DOCKER_TAG}): Failed deploying to staging marathon - ${env.BUILD_URL}")
            }
          }
        }
      }
    }
  }
}
