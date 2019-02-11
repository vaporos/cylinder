#!groovy

// Copyright 2017 Intel Corporation
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//     http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.
// ------------------------------------------------------------------------------

// Discard old builds after 31 days
properties([[$class: 'BuildDiscarderProperty', strategy:
        [$class: 'LogRotator', artifactDaysToKeepStr: '',
        artifactNumToKeepStr: '', daysToKeepStr: '31', numToKeepStr: '']]]);

node ('master') {
    timestamps {
        // Create a unique workspace so Jenkins doesn't reuse an existing one
        ws("workspace/${env.BUILD_TAG}") {
            stage("Clone Repo") {
                checkout scm
            }

            if (!(env.BRANCH_NAME == 'master' && env.JOB_BASE_NAME == 'master')) {
                stage("Check Whitelist") {
                    readTrusted 'bin/whitelist'
                    sh './bin/whitelist "$CHANGE_AUTHOR" /etc/jenkins-authorized-builders'
                }
            }

            stage("Check for Signed-Off Commits") {
                sh '''#!/bin/bash -l
                    if [ -v CHANGE_URL ] ;
                    then
                        temp_url="$(echo $CHANGE_URL |sed s#github.com/#api.github.com/repos/#)/commits"
                        pull_url="$(echo $temp_url |sed s#pull#pulls#)"

                        IFS=$'\n'
                        for m in $(curl -s "$pull_url" | grep "message") ; do
                            if echo "$m" | grep -qi signed-off-by:
                            then
                              continue
                            else
                              echo "FAIL: Missing Signed-Off Field"
                              echo "$m"
                              exit 1
                            fi
                        done
                        unset IFS;
                    fi
                '''
            }

            // Set the ISOLATION_ID environment variable for the whole pipeline
            env.ISOLATION_ID = sh(returnStdout: true, script: 'printf $BUILD_TAG | sha256sum | cut -c1-64').trim()
            env.COMPOSE_PROJECT_NAME = sh(returnStdout: true, script: 'printf $BUILD_TAG | sha256sum | cut -c1-64').trim()

            // Use a docker container to build and protogen, so that the Jenkins
            // environment doesn't need all the dependencies.

            stage("Build Lint Requirements") {
                sh 'docker-compose -f docker/compose/run-lint.yaml build'
                sh 'docker-compose -f docker/compose/sawtooth-build.yaml up'
                sh 'docker-compose -f docker/compose/sawtooth-build.yaml down'
            }

            stage("Run Lint") {
                sh 'docker-compose -f docker/compose/run-lint.yaml up --abort-on-container-exit --exit-code-from lint-rust lint-rust'
            }

            // Run the tests
            stage("Run Tests") {
                sh 'INSTALL_TYPE="" ./bin/run_tests'
            }

            stage("Build documentation") {
                sh 'docker build . -f docs/Dockerfile -t sawtooth-sdk-rust-docs:$ISOLATION_ID'
                sh 'docker run --rm -v $(pwd):/project/sawtooth-sdk-rust sawtooth-sdk-rust-docs:$ISOLATION_ID'
            }

            stage("Archive Build artifacts") {
                archiveArtifacts artifacts: 'docs/build/**'

            }
        }
    }
}