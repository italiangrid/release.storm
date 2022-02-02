#!/usr/bin/env groovy

def platform2Repo = [
  "centos7" : "centos7",
  "centos7java11" : "centos7",
  "centos6": "centos6"
]

def buildRepoName(repo, platform) {

  def repoName
  if (platform ==~ /^centos\d+.*/) {
    repoName = "${repo}-rpm-${env.BRANCH_NAME}"
  } else if (platform ==~ /^ubuntu\d+/) {
    repoName = "${repo}-deb-${env.BRANCH_NAME}-${platform}"
  } else {
    error("Unsupported platform: ${platform}")
  }
  echo "Repo name: $repoName"
  return repoName
}

def removePackages(repo, platform, platform2Repo) {

  def platformRepo = platform2Repo[platform]
  if (!platformRepo) {
    error("Unknown platform: ${platform}")
  }
  echo "platformRepo = $platformRepo"

  if (platform ==~ /^centos\d+.*/) {
    sh "nexus-assets-remove -u ${env.NEXUS_CRED_USR} -p ${env.NEXUS_CRED_PSW} -H ${env.NEXUS_HOST} -r ${repo} -q ${platformRepo}"
  } else if (platform ==~ /^ubuntu\d+/) {
    sh "nexus-assets-remove -u ${env.NEXUS_CRED_USR} -p ${env.NEXUS_CRED_PSW} -H ${env.NEXUS_HOST} -r ${repo} -q packages"
    sh "nexus-assets-remove -u ${env.NEXUS_CRED_USR} -p ${env.NEXUS_CRED_PSW} -H ${env.NEXUS_HOST} -r ${repo} -q metadata"
  } else {
    error("Unsupported platform: ${platform}")
  }
}

def publish(repo, platform, platform2Repo) {

  def platformRepo = platform2Repo[platform]
  if (!platformRepo) {
    error("Unknown platform: ${platform}")
  }
  echo "platformRepo = $platformRepo"

  if (platform ==~ /^centos\d+.*/) {
    sh "nexus-assets-flat-upload -u ${env.NEXUS_CRED_USR} -p ${env.NEXUS_CRED_PSW} -H ${env.NEXUS_HOST} -r ${repo}/${platformRepo} -d artifacts/packages/${platform}/RPMS"
  } else if (platform ==~ /^ubuntu\d+/) {
    sh "nexus-assets-flat-upload -f -u ${env.NEXUS_CRED_USR} -p ${env.NEXUS_CRED_PSW} -H ${env.NEXUS_HOST} -r ${repo} -d artifacts/packages/${platform}"
  } else {
    error("Unsupported platform: ${platform}")
  }
}

def doPlatform(repo, platform, platform2Repo) {
  return {
    def repoName = buildRepoName(repo, platform)

    if (env.CLEANUP_REPO) {
      removePackages(repoName, platform, platform2Repo)
    }

    publish(repoName, platform, platform2Repo)
  }
}

def publishPackages(releaseInfo,platform2Repo) {

  copyArtifacts filter: 'artifacts/**', fingerprintArtifacts: true, projectName: "${releaseInfo.project}", target: '.'

  def repo = releaseInfo.repo
  def platforms = releaseInfo.platforms

  echo "Publish for the following ${platforms}"

  def publishStages = platforms.collectEntries {
    [ "${env.BRANCH_NAME} - ${it}" : doPlatform(repo,it,platform2Repo) ]
  }

  parallel publishStages
}

def doPublish(platform2Repo) {
  def releaseInfo = readYaml file:'release-info.yaml'
  publishPackages(releaseInfo,platform2Repo)
}

def isBranchIndexingCause() {
  def isBranchIndexing = false
  if (!currentBuild.rawBuild) {
    return true
  }
  currentBuild.rawBuild.getCauses().each { cause ->
    if (cause instanceof jenkins.branch.BranchIndexingCause) {
      isBranchIndexing = true
    }
  }
  return isBranchIndexing
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
          doPublish(platform2Repo)
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
          doPublish(platform2Repo)
        }
      }
    }

    stage('publish packages (stable)') {

      when {
        branch 'stable'
      }

      steps {
        script {
          if (!isBranchIndexingCause()) {
            doPublish(platform2Repo)
          }
        }
      }
    }
  }
}
