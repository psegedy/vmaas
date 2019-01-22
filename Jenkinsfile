/*
 * Pipeline for running integration tests in Jenkins job with 
 * every pull request, change in master and every 24h
 *
 * Requires: https://github.com/RedHatInsights/insights-pipeline-lib
 */

@Library("github.com/RedHatInsights/insights-pipeline-lib") _

// Code coverage failure threshold
codecovThreshold = 89

properties(
    [
        pipelineTriggers([cron('H 22 * * *')]),
    ]
)

node {
    // Cancel any prior builds that are running for this job
    cancelPriorBuilds()

    runStages()
}


def runStages() {
    // withNode is a helper to spin up a jnlp slave using the Kubernetes plugin, and run the body code on that slave
    openShift.withNode(image: "docker-registry.default.svc:5000/jenkins/jenkins-slave-vmaas:latest") {
        // check out source again to get it in this node's workspace
        scmVars = checkout scm

        // checkout vmaas-tests git repository
        checkOutRepo targetDir: 'vmaas-tests', repoUrl: 'https://github.com/RedHatInsights/vmaas-tests'
        // checkOutRepo targetDir: 'vmaas-yamls', repoUrl: 'https://github.com/psegedy/vmaas-yamls'

        stage('Pip install') {
            withStatusContext.pipInstall {
                // install devpi
                sh 'pip install --user --no-warn-script-location -U pip devpi-client setuptools setuptools_scm wheel'
                // set devpi address
                sh "devpi use http://devpi.devpi.svc/root/psav --set-cfg"
                // install iqe-tests
                sh "pip install --user --no-warn-script-location iqe-integration-tests iqe-clientv3-plugin"
                // install vulnerability plugin
                // sh "pip install --user --no-warn-script-location iqe-vulnerability-plugin"
                sh "pip install --user --no-warn-script-location -e vmaas-tests"
                sh "pip install --user --no-warn-script-location pytest-html"
                sh "pip install --user --no-warn-script-location pytest-report-parameters"
            }
        }

        // stage('Deploy vmaas') {
        //     // todo With status context
        //     stage('Login as deployer account') {
        //         withCredentials([string(credentialsId: 'envConfig['deployerSecretId']', variable: 'TOKEN')]) {
        //             sh "oc login https://${pipelineVars.devCluster} --token=${TOKEN}"
        //         }

        //         sh "oc project ${envConfig['project']}"
        //     }
        //     deployService(service: 'templates/vmaas', project: 'vmaas-qe', env: 'ci')
        // }

        stage('Integration tests') {
            // withStatusContext runs the body code and notifies GitHub on whether it passed or failed
            // 'unitTest' will notify the "continuous-integration/jenkins/unittest" status
            sh "${pipelineVars.userPath}/iqe plugin list"
            withStatusContext.unitTest {
                sh "cd ${WORKSPACE} && pwd"
                sh "echo 'RUN TESTS' | tee junit.xml"
            }
            junit 'junit.xml'
        }

        stage('Code coverage') {
            // Checks code coverage results of the above unit tests with coverage.py, this step fails if coverage is below codecovThreshold
            // Notifies GitHub with the "continuous-integration/jenkins/coverage" status
            checkCoverage(threshold: codecovThreshold)
        }

        if (currentBuild.currentResult == 'SUCCESS') {
            if (env.BRANCH_NAME == 'master') {
                // Stages to run specifically if master branch was updated
                // rebuild and deploy vmaas
            }
        }
    }
}