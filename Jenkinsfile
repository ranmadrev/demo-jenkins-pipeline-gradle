pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                checkout scm
                sh "env"
                sh "gradle bootRepackage --stacktrace"
                sh 'jarFile=`ls build/libs | grep -v original` && mkdir -p ocp/deployments && cp build/libs/$jarFile ocp/deployments/'
            }
        }
        stage('Test') {
            steps {
                parallel(
                    'check': {
                        sh "gradle check --stacktrace"
                    },
                    'echo': {
                        echo "ok in parallel"
                    }
                )
            }
        }
        stage('Build image') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject('dev') {
                            // create docker build
                            def buildTemplate = readYaml file: 'openshift/demo-docker-build.yml'
                            def resources = openshift.process(buildTemplate)
                            def buildCfg = openshift.apply(resources).narrow('bc')
                            // create app template
                            def template = readYaml file: 'openshift/demo-template.yml'
                            openshift.apply(openshift.process(template))

                            def buildSelector = buildCfg.startBuild('--from-dir=ocp')
                            timeout(5) {
                                buildSelector.untilEach(1) {
                                    return it.object().status.phase == "Complete"
                                }
                            }
                        }
                    }
                }
            }
        }
        stage('Dev') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject('dev') {
                            // TODO: replace it when https://github.com/openshift/jenkins-client-plugin/issues/84 will be solved
                            // deployment is triggered by imagestream here
                            openshiftVerifyDeployment depCfg: "demo", namespace: "dev"
                            sh "curl -L http://demo.dev.svc.cluster.local:8080/health"
                        }
                    }
                }
            }
        }
        stage('Staging') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.tag("dev/demo:latest", "staging/demo:latest")
                        openshift.withProject('staging') {
                            def template = readYaml file: 'openshift/demo-template.yml'
                            openshift.apply(openshift.process(template, "-p", "VERSION=latest"))
                            // TODO: replace it when https://github.com/openshift/jenkins-client-plugin/issues/84 will be solved
                            openshiftVerifyDeployment depCfg: "demo", namespace: "staging"
                        }
                    }
                }
            }
        }
        stage('Approve') {
            agent none
            steps {
                input "Does the staging environment look ok ?"
            }
        }
        stage('Prod') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.tag("staging/demo:latest", "prod/demo:latest")
                        openshift.withProject('prod') {
                            def template = readYaml file: 'openshift/demo-template.yml'
                            openshift.apply(openshift.process(template, "-p", "VERSION=latest"))
                            // TODO: replace it when https://github.com/openshift/jenkins-client-plugin/issues/84 will be solved
                            openshiftVerifyDeployment depCfg: "demo", namespace: "prod"
                        }
                    }
                }
            }
        }
    }
}
