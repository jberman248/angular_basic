def shipyardBuildBadge = addEmbeddableBadgeConfiguration(id: "shipyard-build", subject: "Shipyard Build")

pipeline {
    agent {
        node {
            label 'ubuntu-slave'
        }
    }

    environment {
        EMAIL_RECIPIANTS = 'ljolliff@cynerge.com'
        NEXUS_USER = credentials('nexus-user')
        NEXUS_PASS = credentials('nexus-pass')
        NEXUS_REPO = credentials('nexus-raw-repo')
        APP_SOURCE = './src/**/**/**/**.html'
        STATUS_SUCCESS = ''
        JENKINS_URL = "${JENKINS_URL}"
        JOB_NAME = "${JOB_NAME}"
      //  SONAR_TOKEN = credentials('govcloud-sonarqube')
      //  SONAR_PROJECT = 'shipyard-project'
      //  SONAR_SOURCE = 'src'
    }

    stages {

        stage('Dependencies') {
            agent {
                docker {
                    image 'luther007/cynerge_images'
                    args '-u root'
                    alwaysPull true
                }
            }
            steps {
                echo 'Installing...'
                sh 'echo $GIT_BRANCH'
                sh 'npm ci'
            }
        }

        // Run accessability scanning with Pa11y
        stage('Pa11y') {
            agent {
                docker {
                    image 'luther007/cynerge_images'
                    args '-u root'
                    alwaysPull true
                }
            }

            steps {
                sh 'rm -rf test-results || echo "directory does not exist"'
                sh 'mkdir test-results'
                sh 'chmod -R 777 test-results/'
                sh "pa11y-ci -T 5 ${env.APP_SOURCE} --json > test-results/pa11y-ci-results.json"
                dir('test-results') {
                    sh 'pa11y-ci-reporter-html'
                }
            }
            post {
                success {
                    // Do NOT delete the empty line underneath below curl command. It is necessary for script logic
                    dir('test-results') {
                        sh "curl -v --user '${NEXUS_USER}:${NEXUS_PASS}' --upload-file \"{\$(echo *.html | tr ' ' ',')}\" ${NEXUS_REPO}Pa11y/${JOB_NAME}/${BRANCH_NAME}/${BUILD_NUMBER}/"

                    }
                }
            }

        }
      
    cleanup {
        script {
            shipyardBuildBadge.setStatus('running')
            try {
                    shipyardBuildBadge.setStatus('passing')
                } catch (Exception err) {
                    shipyardBuildBadge.setStatus('failing')
                    shipyardBuildBadge.setColor('red')

                    error 'Build failed'
                }

            cleanWs()
        }
    }
    }
}
