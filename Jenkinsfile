pipeline {
  agent none
  stages {
    /** - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
     * Build in parallel
    **/
    stage("build stages") {
      parallel {
        stage("build") {
          agent {
            // ruby:2.5 is gitlab default
            docker { image 'ruby:2.5' }
          }
          steps {
            sh 'echo "This is some text" > my_file.txt'
            archiveArtifacts artifacts: 'my_file.txt'
          }
        }

        stage("kaniko build") {
          agent {
            docker {
              image 'gcr.io/kaniko-project/executor:debug'
              args '--entrypoint ""'
            }
          }
          steps {
            withCredentials([usernamePassword(credentialsId: 'dockerRegistry', usernameVariable: 'CI_REGISTRY_USER', passwordVariable: 'CI_REGISTRY_PASSWORD')]) {
              sh """
                export CI_REGISTRY='registry.gitlab.com'
                export CI_PROJECT_DIR='.'
                export CI_COMMIT_SHORT_SHA="\$(echo \$GIT_COMMIT | head -c8)"
                export CI_PROJECT_PATH_SLUG="\$(echo \$GIT_URL | sed 's;https://gitlab.com;;g' | sed 's;\\.git;;g' | sed 's;\\/;-;g')"
                export CI_REGISTRY_IMAGE="\$CI_REGISTRY/\$CI_PROJECT_PATH_SLUG"

                mkdir -p /kaniko/.docker
                echo "{\"auths\":{\"\$CI_REGISTRY\":{\"username\":\"\$CI_REGISTRY_USER\",\"password\":\"\$CI_REGISTRY_PASSWORD\"}}}" > /kaniko/.docker/config.json
                /kaniko/executor \\\
                    --context \$CI_PROJECT_DIR \\\
                    --dockerfile \$CI_PROJECT_DIR/Dockerfile \\\
                    --destination \$CI_REGISTRY_IMAGE:build-\$CI_COMMIT_SHORT_SHA \\\
                    --build-arg CI_BUILD_ID=\$CI_BUILD_ID \\\
                    --build-arg CI_PROJECT_PATH_SLUG=\$CI_PROJECT_PATH_SLUG
              """
            }
          }
        }
      }
    }// END build

    /** - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
     * Test in parallel
    **/
    stage('test stages') {
      parallel {
        stage("env") {
          agent {
            // ruby:2.5 is gitlab default
            docker { image 'ruby:2.5' }
          }
          steps {
            sh 'env'
          }
        }

        stage("test") {
          agent {
            docker { image 'ruby:2.5' }
          }
          steps {
            // Requires plugin
            ansiColor('xterm') {
              sh """
                TXT_RED="\\e[31m" && TXT_CLEAR="\\e[0m"
                echo "download stuff"
                echo "dummy test"
                echo -e "\${TXT_RED}This text is red,\${TXT_CLEAR} but this part isn't\${TXT_RED} however this part is again."
              """
            }
          }
        }
      }
    } // END test

    /** - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
     * Staging deploy with timeout
    **/
    stage('staging deploy') {
      agent {
        docker { image 'ruby:2.5' }
      }
      steps{
        timeout(time: 10, uint: 'MINUTES') {
          sh """
            echo "staging deploy"
          """
        }
      }
    } // END staging deploy

    /** - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
     * Prod deploy with user input
    **/
    stage('production deploy') {
      agent {
        docker { image 'ruby:2.5' }
      }
      steps{
        input {
          message "Should we continue?"
          ok "yes"
          parameters {
            string(name: 'CONTINUE', defaultValue: 'no', description: 'Should we continue?')
          }
        }
        sh """
          if [ "${CONTINUE}" == "yes" ]; then
            echo "production deploy"
            echo "dummy validate production deploy"
          fi
        """
      }
    }
  } // END prod deploy

  post {
    success {
      // Need a node since agent none
      node('master') {
        sh """
          echo "notify success"
        """
      }
    }
    failure {
      node('master') {
        sh """
          echo "notify failure"
        """
      }
    }
  }
}