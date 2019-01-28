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
        checkOutRepo(targetDir: "vmaas_tests", repoUrl: "https://github.com/RedHatInsights/vmaas_tests", credentialsId: "github")
        checkOutRepo(targetDir: "vmaas-yamls", repoUrl: "https://github.com/psegedy/vmaas-yamls", credentialsId: "github")

        // if (currentBuild.currentResult == "SUCCESS") {
        //     if (env.BRANCH_NAME == "master") {
        //         // Stages to run specifically if master branch was updated
        //         // rebuild and deploy vmaas
        //     }
        // }

        stage("Pip install") {
            withStatusContext.pipInstall {
                // install devpi
                sh "pip install --user --no-warn-script-location -U pip devpi-client setuptools setuptools_scm wheel"
                // set devpi address
                sh "devpi use ${DEV_PI} --set-cfg"
                // install iqe-tests
                sh "pip install --user --no-warn-script-location iqe-integration-tests iqe-clientv3-plugin iqe-current-ui-plugin"
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

            checkOutRepo(targetDir: pipelineVars.e2eDeployDir, "https://github.com/psegedy/e2e-deploy", credentialsId: "github")
            sh "python3.6 -m venv ${pipelineVars.venvDir}"
            sh "${pipelineVars.venvDir}/bin/pip install --upgrade pip"
            dir(pipelineVars.e2eDeployDir) {
                sh "${pipelineVars.venvDir}/bin/pip install -r requirements.txt"
                // wipe old deployment
                sh "${pipelineVars.venvDir}/bin/ocdeployer wipe -f vmaas-qe -l app=vmaas"
                sh """
                    git checkout vmaas_qa_needs
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
                    echo "  DOCKERFILE_PATH: Dockerfile-qe" >> builder-env.yml
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
                    echo "  RESTART_POLICY: Never" >> env.yml
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
                sh """
                    cd ~/.local/lib/python3.6/site-packages/iqe/conf
                    ln -rs ${WORKSPACE}/vmaas-yamls/conf/credentials.yaml
                """
                sh """
                    cd vmaas_tests/iqe_vulnerability/conf
                    ln -rs ${WORKSPACE}/vmaas-yamls/conf/settings.local.yaml
                    sed -i 's|localhost|http://vmaas-webapp.vmaas-qe.svc:8080|g' settings.default.yaml
                """
                stage("Run vmaas-webapp with coverage") {
                    sh '''
                        webapp_pod="$(oc get pods | grep 'Running' | grep 'webapp' | awk '{print $1}')"
                        oc exec "${webapp_pod}" -- bash -c "coverage run /app/app.py --source app &>/proc/1/fd/1"
                    '''
                }
                stage("Setup DB") {
                    withCredentials([string(credentialsId: "vmaas-bot-token", variable: "TOKEN")]) {
                    sh """
                        cd vmaas_tests
                        vmaas/scripts/setup_db.sh ${WORKSPACE}/vmaas-yamls/data/repolist.json \
                            http://vmaas-reposcan.vmaas-qe.svc:8081 \
                            http://vmaas-webapp.vmaas-qe.svc:8080 \
                            ${TOKEN}
                        sleep 10
                    """   
                    }
 
                }
                stage("Run tests") {
                    sh '''
                        cd vmaas_tests
                        pytest_status=0
                        # run only vmaas tests for now
                        if [ -d iqe_vulnerability/tests/vmaas ]; then
                            ~/.local/bin/iqe tests custom -v --junit-xml="iqe-junit-report.xml" --html="report.html" --self-contained-html iqe_vulnerability/tests/vmaas || pytest_status="$?"
                        else
                            ~/.local/bin/iqe tests plugin vulnerability -v --junit-xml="iqe-junit-report.xml" --html="report.html" --self-contained-html || pytest_status="$?"
                        fi
                        # Running pytest can result in six different exit codes:
                        # 0: All tests were collected and passed successfully
                        # 1: Tests were collected and run but some of the tests failed
                        # 2: Test execution was interrupted by the user
                        # 3: Internal error happened while executing tests
                        # 4: pytest command line usage error
                        # 5: No tests were collected
                        if [ "$pytest_status" -gt 1 ]; then
                            exit "$pytest_status"
                        fi
                    '''
                }
            }
            junit "vmaas_tests/iqe-junit-report.xml"

        stage("Code coverage") {
            // Notifies GitHub with the "continuous-integration/jenkins/coverage" status
            // Kill webapp and copy .coverage file
            sh '''
                webapp_pod="$(oc get pods | grep 'Running' | grep 'webapp' | awk '{print $1}')"
                oc exec "${webapp_pod}" -- pkill -sigint coverage
                oc cp "${webapp_pod}":/tmp/.coverage .coverage
            '''
            def status = 99

            status = sh(
                script: "${pipelineVars.userPath}/coverage html --fail-under=${threshold} --omit /usr/*",
                returnStatus: true
            )

            archiveArtifacts 'htmlcov/*'

            withStatusContext.coverage {
                assert status == 0
            }
        }
    }
}
