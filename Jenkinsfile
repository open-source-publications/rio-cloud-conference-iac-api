pipeline{

    agent{
        // label "" also could have been 'agent any' - that has the same meaning.
        label "master"
    }

    environment {
        // Global Vars
        NAMESPACE_PREFIX="conference"

        PIPELINES_NAMESPACE = "${NAMESPACE_PREFIX}-ci-cd"
        APP_NAME = "java-app"

        JENKINS_TAG = "${JOB_NAME}.${BUILD_NUMBER}".replace("/", "-")
        JOB_NAME = "${JOB_NAME}".replace("/", "-")

        GIT_SSL_NO_VERIFY = true

        DISPLAY = 0

        pom = readMavenPom file: 'pom.xml'

        ARTIFACTID = pom.getArtifactId();
        VERSION = pom.getVersion();

        RELEASE = false
    }

    // The options directive is for configuration that applies to the whole job.
    options {
        buildDiscarder(logRotator(numToKeepStr:'10'))
        timeout(time: 15, unit: 'MINUTES')
        ansiColor('xterm')
        timestamps()
    }

    stages{
        stage("Prepare environment for master deploy") {
            agent {
                node {
                    label "master"
                }
            }
            when {
              expression { GIT_BRANCH ==~ /(.*master)/ }
            }
            steps {
                script {
                    // Arbitrary Groovy Script executions can do in script tags
                    env.PROJECT_NAMESPACE = "${NAMESPACE_PREFIX}-test"
                    env.NODE_ENV = "test"
                    env.RELEASE = true
                }
            }
        }
        stage("Prepare environment for develop deploy") {
            agent {
                node {
                    label "master"
                }
            }
            when {
              expression { GIT_BRANCH ==~ /(.*develop|.*feature.*)/ }
            }
            steps {
                script {
                    // Arbitrary Groovy Script executions can do in script tags
                    env.PROJECT_NAMESPACE = "${NAMESPACE_PREFIX}-dev"
                    env.NODE_ENV = "dev"
                }
            }
        }
        
        stage("Test/Build/Nexus/OpenShift Build"){
            agent {
                node {
                    label "maven"
                }
            }
            stages{
                stage("Print ArtifactID and Version"){
                    steps{
                        script{
                            // Arbitrary Groovy Script executions can do in script tags
                            echo "Version ::: ${VERSION}"
                            echo "artifactId ::: ${ARTIFACTID}"
                        }
                    }
                }
                stage("Test Project"){
                    steps {
                        sh 'printenv'

                        echo '### Running tests ###'
                        sh 'mvn clean test'
                    }
                }
                stage("Build Project"){
                    steps{
                        echo '### Running install ###'
                        sh 'mvn clean install'
                    }
                }
                /*
                stage("Verify with Sonar"){
                    steps{
                        echo '### Maven Verify Sonar ###'
                        sh 'mvn pmd:pmd checkstyle:checkstyle'
                        sh 'mvn org.jacoco:jacoco-maven-plugin:prepare-agent install -Dmaven.test.failure.ignore=true -Pcoverage-per-test'
                        
                        withSonarQubeEnv('sonar') {
                            echo '### Running static code analysis ###'
                            sh 'mvn org.jacoco:jacoco-maven-plugin:report-aggregate'
                            sh 'mvn sonar:sonar'
                        }
                    }
                }
                stage("Sonar Quality Gate"){
                    steps {
                        timeout(time: 1, unit: 'HOURS') {
                            waitForQualityGate abortPipeline: false
                        }
                    }
                }
                */
                
                //stage("Deploy to Nexus"){
                  //  when {
                      //  expression { currentBuild.result != 'UNSTABLE' }
                    //}
                    //steps{
                      //  echo '### Running deploy ###'
                     //   sh 'mvn deploy'
                   // }
                    // Post can be used both on individual stages and for the entire build.
                   // post {
                     //   always{
                       //     archive "**"
                //            junit testResults: '**/target/surefire-reports/TEST-*.xml'

                            // Notify slack or some such
                        //}
                       // success {
                        //    echo "Git tagging"
                         //   sh'''
                          //      git config --global user.email "svc_via_varejo@jmail.com"
                           //     git config --global user.name "svc_via_varejo"
                            //    git tag -a ${JENKINS_TAG} -m "JENKINS automated commit"
                             //   git push http://${GIT_CREDENTIALS_USR}:${GIT_CREDENTIALS_PSW}@${GIT_DOMAIN}/${GIT_ORG}/${APP_NAME}.git --tags
                            //'''
                       // }
                        //failure {
                         //   echo "FAILURE"
                        //}
                    //}
               // }
                
                stage("Start Openshift Build"){
                    when {
                        expression { currentBuild.result != 'UNSTABLE' }
                    }
                    steps{
                        echo '### Create Linux Container Image from package ###'
                        sh  '''
                                oc project ${PIPELINES_NAMESPACE} # probs not needed
                                oc patch bc ${APP_NAME} -p "{\\"spec\\":{\\"output\\":{\\"to\\":{\\"kind\\":\\"ImageStreamTag\\",\\"name\\":\\"${APP_NAME}:${JENKINS_TAG}\\"}}}}"
                                oc start-build ${APP_NAME} --from-file=target/${ARTIFACTID}-${VERSION}.jar --follow
                            '''
                    }
                }
            }
        }
        /*
        stage("Openshift Deployment") {
            agent {
                node {
                    label "master"
                }
            }
            when {
                allOf{
                    expression { GIT_BRANCH ==~ /(.*master|.*develop|.*feature.*)/ }
                    expression { currentBuild.result != 'UNSTABLE' }
                }
            }
            steps {
                echo '### tag image for namespace ###'
                sh  '''
                    oc project ${PROJECT_NAMESPACE}
                    oc tag ${PIPELINES_NAMESPACE}/${APP_NAME}:${JENKINS_TAG} ${PROJECT_NAMESPACE}/${APP_NAME}:${JENKINS_TAG}
                    '''
                echo '### set env vars and image for deployment ###'
                sh '''
                    oc set env dc ${APP_NAME} NODE_ENV=${NODE_ENV} SPRING_PROFILES_ACTIVE=${SPRING_PROFILES_ACTIVE}
                    oc set image dc/${APP_NAME} ${APP_NAME}=docker-registry.default.svc:5000/${PROJECT_NAMESPACE}/${APP_NAME}:${JENKINS_TAG}
                    oc label --overwrite dc ${APP_NAME} stage=${NODE_ENV}
                    oc patch dc ${APP_NAME} -p "{\\"spec\\":{\\"template\\":{\\"metadata\\":{\\"labels\\":{\\"version\\":\\"${VERSION}\\",\\"release\\":\\"${RELEASE}\\",\\"stage\\":\\"${NODE_ENV}\\",\\"git-commit\\":\\"${GIT_COMMIT}\\",\\"jenkins-build\\":\\"${JENKINS_TAG}\\"}}}}}"
                    oc rollout latest dc/${APP_NAME}
                '''
                echo '### Verify OCP Deployment ###'
                openshiftVerifyDeployment depCfg: env.APP_NAME,
                    namespace: env.PROJECT_NAMESPACE,
                    replicaCount: '1',
                    verbose: 'false',
                    verifyReplicaCount: 'true',
                    waitTime: '',
                    waitUnit: 'sec'
            }
        }*/
    }
}
