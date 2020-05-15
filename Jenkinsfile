#!/usr/bin/env groovy

def buildRepoName(repo, platform) {

  if (platform ==~ /^centos\d+/) {
    return "${repo}-rpm-${env.BRANCH_NAME}"
  } else if (platform ==~ /^ubuntu\d+/) {
    return "${repo}-deb-${env.BRANCH_NAME}-${platform}"
  } else {
    error("Unsupported platform: ${platform}")
  }
}

def removePackages(repo, platform) {

  if (platform ==~ /^centos\d+/) {
    sh "nexus-assets-remove -u ${env.NEXUS_CRED_USR} -p ${env.NEXUS_CRED_PSW} -H ${env.NEXUS_HOST} -r ${repo} -q ${platform}"
  } else if (platform ==~ /^ubuntu\d+/) {
    sh "nexus-assets-remove -u ${env.NEXUS_CRED_USR} -p ${env.NEXUS_CRED_PSW} -H ${env.NEXUS_HOST} -r ${repo} -q packages"
    sh "nexus-assets-remove -u ${env.NEXUS_CRED_USR} -p ${env.NEXUS_CRED_PSW} -H ${env.NEXUS_HOST} -r ${repo} -q metadata"
  } else {
    error("Unsupported platform: ${platform}")
  }
}

def publish(repo, platform) {

  if (platform ==~ /^centos\d+/) {
    sh "nexus-assets-flat-upload -u ${env.NEXUS_CRED_USR} -p ${env.NEXUS_CRED_PSW} -H ${env.NEXUS_HOST} -r ${repo}/${platform} -d artifacts/packages/${platform}/RPMS"
  } else if (platform ==~ /^ubuntu\d+/) {
    sh "nexus-assets-flat-upload -f -u ${env.NEXUS_CRED_USR} -p ${env.NEXUS_CRED_PSW} -H ${env.NEXUS_HOST} -r ${repo} -d artifacts/packages/${platform}"
  } else {
    error("Unsupported platform: ${platform}")
  }
}

def doPlatform(repo, platform) {
  return {
    def repoName = buildRepoName(repo, platform)

    if (env.CLEANUP_REPO) {
      removePackages(repoName, platform)
    }

    publish(repoName, platform)
  }
}

def publishPackages(releaseInfo) {

  copyArtifacts filter: 'artifacts/**', fingerprintArtifacts: true, projectName: "${releaseInfo.project}", target: '.'

  def repo = releaseInfo.repo
  def platforms = releaseInfo.platforms

  echo "Publish for the following ${platforms}"

  def publishStages = platforms.collectEntries {
    [ "${env.BRANCH_NAME} - ${it}" : doPlatform(repo,it) ]
  }

  parallel publishStages
}

def doPublish() {
  def releaseInfo = readYaml file:'release-info.yaml'
  publishPackages(releaseInfo)
}

pipeline {

  agent {
    label 'docker'
  }

  options {
    timeout(time: 10, unit: 'MINUTES')
  }

  environment {
    NEXUS_HOST = "https://repo.cloud.cnaf.infn.it"
    NEXUS_CRED = credentials('jenkins-nexus')
  }

  triggers { cron('@daily') }

  stages {

    stage('checkout') {
      steps {
        deleteDir()
        checkout scm
      }
    }

    stage('publish packages (nightly)') {

      environment {
        CLEANUP_REPO = "y"
      }

      when {
        branch 'nightly'
      }

      steps {
        script {
          doPublish()
        }
      }
    }

    stage('publish packages (beta)') {

      environment {
        CLEANUP_REPO = "y"
      }

      when {
        branch 'beta'
      }

      steps {
        script {
          doPublish()
        }
      }
    }

    stage('publish packages (stable)') {

      when {
        branch 'stable'
      }

      steps {
        script {
          doPublish()
        }
      }
    }
  }
}
