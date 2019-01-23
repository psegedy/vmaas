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
        pipelineTriggers([cron("H 22 * * *")]),
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
        // check out source again to get it in this node"s workspace
        scmVars = checkout scm

        // checkout vmaas_tests git repository
        checkOutRepo(targetDir: "vmaas_tests", repoUrl: "https://github.com/psegedy/vmaas_tests")
        // checkOutRepo targetDir: "vmaas-yamls", repoUrl: "https://github.com/psegedy/vmaas-yamls" credentialsId: ""

        if (currentBuild.currentResult == "SUCCESS") {
            if (env.BRANCH_NAME == "master") {
                // Stages to run specifically if master branch was updated
                // rebuild and deploy vmaas
            }
        }

        stage("Pip install") {
            withStatusContext.pipInstall {
                // install devpi
                sh "pip install --user --no-warn-script-location -U pip devpi-client setuptools setuptools_scm wheel"
                // set devpi address
                sh "devpi use ${DEV_PI} --set-cfg"
                // install iqe-tests
                sh "pip install --user --no-warn-script-location iqe-integration-tests iqe-clientv3-plugin"
                // install vulnerability plugin
                // sh "pip install --user --no-warn-script-location iqe-vulnerability-plugin"
                sh "pip install --user --no-warn-script-location -e vmaas_tests"
                sh "pip install --user --no-warn-script-location pytest-html"
                sh "pip install --user --no-warn-script-location pytest-report-parameters"
            }
        }

        stage("Deploy vmaas") {
            // todo With status context
            stage("Login as deployer account") {
                withCredentials([string(credentialsId: "openshift_token", variable: "TOKEN")]) {
                    sh "oc login https://${pipelineVars.devCluster} --token=${TOKEN}"
                }
                sh "oc project vmaas-qe"
            }

            checkOutRepo(targetDir: pipelineVars.e2eDeployDir, repoUrl: pipelineVars.e2eDeployRepoSsh, credentialsId: pipelineVars.gitSshCreds)
            sh "python3.6 -m venv ${pipelineVars.venvDir}"
            sh "${pipelineVars.venvDir}/bin/pip install --upgrade pip"
            dir(pipelineVars.e2eDeployDir) {
                sh "${pipelineVars.venvDir}/bin/pip install -r requirements.txt"
                sh """
                # Create an env.yaml to have the builder pull from a different branch
                echo "vmaas/vmaas-apidoc:" > builder-env.yml
                echo "  SOURCE_REPOSITORY_REF: ${env.BRANCH_NAME}" >> builder-env.yml
                echo "  SOURCE_REPOSITORY_COMMIT: ${scmVars.GIT_COMMIT}" >> builder-env.yml
                echo "  SOURCE_REPOSITORY_URL: ${scmVars.GIT_URL}" >> builder-env.yml
                echo "vmaas/vmaas-reposcan:" >> builder-env.yml
                echo "  SOURCE_REPOSITORY_REF: ${env.BRANCH_NAME}" >> builder-env.yml
                echo "  SOURCE_REPOSITORY_COMMIT: ${scmVars.GIT_COMMIT}" >> builder-env.yml
                echo "  SOURCE_REPOSITORY_URL: ${scmVars.GIT_URL}" >> builder-env.yml
                echo "vmaas/vmaas-webapp:" >> builder-env.yml
                echo "  SOURCE_REPOSITORY_REF: ${env.BRANCH_NAME}" >> builder-env.yml
                echo "  SOURCE_REPOSITORY_COMMIT: ${scmVars.GIT_COMMIT}" >> builder-env.yml
                echo "  SOURCE_REPOSITORY_URL: ${scmVars.GIT_URL}" >> builder-env.yml
                echo "vmaas/vmaas-websocket:" >> builder-env.yml
                echo "  SOURCE_REPOSITORY_REF: ${env.BRANCH_NAME}" >> builder-env.yml
                echo "  SOURCE_REPOSITORY_COMMIT: ${scmVars.GIT_COMMIT}" >> builder-env.yml
                echo "  SOURCE_REPOSITORY_URL: ${scmVars.GIT_URL}" >> builder-env.yml
                echo "vmaas/vmaas-db:" >> builder-env.yml
                echo "  SOURCE_REPOSITORY_REF: ${env.BRANCH_NAME}" >> builder-env.yml
                echo "  SOURCE_REPOSITORY_COMMIT: ${scmVars.GIT_COMMIT}" >> builder-env.yml
                echo "  SOURCE_REPOSITORY_URL: ${scmVars.GIT_URL}" >> builder-env.yml

                # Deploy these customized builders into 'vmaas-qe' project
                ${pipelineVars.venvDir}/bin/ocdeployer deploy -f --sets vmaas --template-dir buildfactory \
                    -e builder-env.yml vmaas-qe --secrets-local-dir secrets/sanitized

                # Configure your service to look in 'vmaas-qe' for its image, rather than 'buildfactory'
                echo "vmaas/vmaas-apidoc:" > env.yml
                echo "  IMAGE_NAMESPACE: vmaas-qe" >> env.yml
                echo "vmaas/vmaas-reposcan:" >> env.yml
                echo "  IMAGE_NAMESPACE: vmaas-qe" >> env.yml
                echo "vmaas/vmaas-webapp:" >> env.yml
                echo "  IMAGE_NAMESPACE: vmaas-qe" >> env.yml
                echo "vmaas/vmaas-websocket:" >> env.yml
                echo "  IMAGE_NAMESPACE: vmaas-qe" >> env.yml
                echo "vmaas/vmaas-db:" >> env.yml
                echo "  IMAGE_NAMESPACE: vmaas-qe" >> env.yml

                # Deploy the vmaas service set, the insights-advisor-api will be using your custom image.
                ${pipelineVars.venvDir}/bin/ocdeployer deploy -f --sets vmaas -e env.yml vmaas-qe --secrets-local-dir secrets/sanitized
                """
            }
        }

        stage("Integration tests") {
            // withStatusContext runs the body code and notifies GitHub on whether it passed or failed
            // "unitTest" will notify the "continuous-integration/jenkins/unittest" status
            sh "${pipelineVars.userPath}/iqe plugin list"
            withStatusContext.unitTest {
                sh "cd ${WORKSPACE} && pwd"
                sh "echo "RUN TESTS" | tee junit.xml"
            }
            junit "junit.xml"
        }

        stage("Code coverage") {
            // Checks code coverage results of the above unit tests with coverage.py, this step fails if coverage is below codecovThreshold
            // Notifies GitHub with the "continuous-integration/jenkins/coverage" status
            checkCoverage(threshold: codecovThreshold)
        }
    }
}