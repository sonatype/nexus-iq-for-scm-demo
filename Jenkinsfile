pipeline {
  agent {
    docker {
      // Docker image with node and git installed
      image 'tarampampam/node:13-alpine'
    }
  }
  stages {
    stage('Build') {
      steps {
        sh 'npm install'
      }
    }
    stage('Test') {
      steps {
        echo "Run your acceptance tests to ensure that IQ remediation changes function as expected in the application"
      }
    }
    stage('Policy Evaluation') {
      // Policy evaluation should only take place against the branch we intend to merge to
      when { branch 'main' }
      steps {
        sh 'npm run build'  // build script using webpack and the copy-modules-webpack-plugin for easy scanning
        nexusPolicyEvaluation iqStage: 'build', iqApplication: 'npm-example',
            iqScanPatterns: [[scanPattern: 'webpack-modules']],
            failBuildOnNetworkError: true
      }
    }
    stage('Incorporate IQ remediation changes to package-lock.json') {
      when {
        // Only work with branches we recognize as created by Nexus IQ
        branch pattern: ".*-to-.*", comparator: "REGEXP"
      }
      steps {
        script {
          sh 'git status'
          // Look to see if package-lock.json file was updated as part of the build process
          def dirty = sh(script: 'git diff HEAD package-lock.json', returnStdout: true).trim().size() > 0
          if (dirty) {
            echo 'detected changes to package-lock.json, merging now'
            withCredentials([
                [$class          : 'UsernamePasswordMultiBinding',
                 credentialsId   : 'github',
                 usernameVariable: 'GITHUB_USERNAME', passwordVariable: 'GITHUB_PASSWORD'
                ]
            ]) {
              // Configure committer details 
              sh 'git config --global user.email sonatype-ci@sonatype.com'
              sh 'git config --global user.name Sonatype CI'
              
              // Only necessary for https urls, can use other mechanisms to authenticate with github as desired
              def updatedUrl = env.GIT_URL.replaceAll('github.com', "${GITHUB_USERNAME}:${GITHUB_PASSWORD}@github.com")
              sh "git config remote.origin.url ${updatedUrl}"
              
              // By default Jenkins will only checkout the specific branch so we configure to pull the branch we 
              // desire to merge to
              sh "git config remote.origin.fetch '+refs/heads/*:refs/remotes/origin/*'"
              sh 'git fetch origin main'
              
              // Commit changes to package-lock.json file, merge and push to the remote branch
              sh 'git commit -m "committing IQ remediation changes" package-lock.json'
              sh 'git checkout main'
              sh "git merge ${GIT_BRANCH}"

              // This should automatically close the open PR when pushed              
              sh 'git push origin main'
            }
          }
          else {
            echo 'no changes detected in package-lock.json, skipping merge'
          }
        }
      }
    }
  }
}
