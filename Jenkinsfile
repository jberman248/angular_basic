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
        SONAR_TOKEN = credentials('govcloud-sonarqube')
        SONAR_PROJECT = 'shipyard-project'
        SONAR_SOURCE = 'src'
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

        stage('Sonarqube Analysis') {
            agent {
                node {
                    label 'master'
                }
            }

            environment {
                scannerHome = tool 'cynerge-sonarqube'
            }
            steps {
                withSonarQubeEnv('Cynerge Sonarqube') {
                    sh 'printenv'
                    sh 'ls'
                    // sh "${JENKINS_HOME}/tools/hudson.plugins.sonar.SonarRunnerInstallation/cynerge-sonarqube/bin/sonar-scanner"
                    sh "${scannerHome}/bin/sonar-scanner -Dsonar.login=$SONAR_TOKEN -Dsonar.projectKey=$SONAR_PROJECT -Dsonar.sources=$SONAR_SOURCE"
                }
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

        stage('Store NPM Artifact') {
            agent {
                docker {
                    image 'luther007/cynerge_images'
                    args '-u root'
                    alwaysPull true
                }
            }
            environment { 
                NPM_USER = credentials('nexus-user')
                NPM_PASS = credentials('nexus-pass')
                NPM_REGISTRY = credentials('nexus-repo')
                NPM_EMAIL = 'npm@site.com'
                NPM_RC_PATH = "${env.WORKSPACE}/.npmrc"
            }
            steps {
                sh 'npm install -g npm-cli-login'
                sh 'touch .npmrc'
                sh 'printenv'
                sh 'npm-cli-login'
                sh 'cat .npmrc'
                sh "npm publish --registry=${env.NPM_REGISTRY}/"
            }
        }

        // Run accessability scanning with Lighthouse
        stage('Lighthouse') {
            agent {
                docker {
                    image 'luther007/cynerge_images'
                    args '-u root'
                    alwaysPull true
                }
            }
            steps {
                sh 'rm -rf report/lighthouse || echo "directory does not exist"'
                sh 'mkdir -p report/lighthouse'
                sh 'chmod -R 777 report/'
                sh 'lighthouse-batch -v -h -f ./sites/sites.txt'
            }
            post {
                success {
                    // Do NOT delete the empty line underneath below curl command. It is necessary for script logic
                    dir('./report/lighthouse') {
                        sh "curl -v --user '${NEXUS_USER}:${NEXUS_PASS}' --upload-file \"{\$(echo *.html | tr ' ' ',')}\" ${NEXUS_REPO}Lighthouse/${JOB_NAME}/${BRANCH_NAME}/${BUILD_NUMBER}/"

                    }
                }
            }
        }
    }

    post {
        success {
            script {
                env.STATUS_SUCCESS = 'Job Complete!'
                env.JENKINS_URL = "${JENKINS_URL}"
                env.JOB_NAME = "${JOB_NAME}"

            sh 'printenv'

            // emailext body: '''${SCRIPT, template="groovy-html.template"}''',
            emailext body: '''${SCRIPT, template="email_report.template"}''',
            mimeType: 'text/html',
            subject: '$PROJECT_NAME - Build # $BUILD_NUMBER - $BUILD_STATUS!',
            to: "${EMAIL_RECIPIANTS}"

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