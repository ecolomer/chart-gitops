pipeline {
    agent {
        label 'ops-alpine'
    }
    options {
        disableConcurrentBuilds()
        timestamps()
        ansiColor('xterm')
    }
    stages {
        stage('chart validation') {
            when {
                not { branch 'master' }
                expression {
                    // Skip if no chart files modified
                    def retVal = sh(returnStatus: true, script: "git diff-tree --no-commit-id --name-only -r ${GIT_COMMIT} | grep -e '\\(\\.yaml\\|\\.tpl\\)\$'")
                    if (retVal) { println "Info: No chart files modified. Skipping stage." }
                    return !retVal
                }
            }
            steps {
                container('ops-alpine') {
                    script {
                        def result = sh (returnStdout:true, script:"""
                            ROOT_DIR=\$PWD

                            # Process all modified chart templates
                            for dir in \$(git diff-tree --no-commit-id --name-only -r ${GIT_COMMIT} | sed -ne '/\\(\\.yaml\\|\\.tpl\\)\$/p' | sed -e 's|\\([^/]\\+/[^/]\\+\\)/.*|\\1|' | uniq); do
                                cd \$dir

                                (set +x; echo "\n[1mLinting \$dir ...[0m")
                                if ! helm lint; then
                                    (set +x; echo "[31mError: chart issues found! Review error messages.[0m")
                                    break
                                fi

                                cd \$ROOT_DIR
                            done
                        """)

                        // Display script output
                        println "\n\u001B[1mChart validation stage output\u001B[0m\n$result"

                        // If issues were found, fail build
                        if (result.contains("Error:")) { currentBuild.result = "FAILURE"; }
                    }
                }
            }
        }
        stage('chart package/publish') {
            when {
                branch 'master'
                expression {
                    // Skip if no chart files modified
                    def retVal = sh(returnStatus: true, script: "git diff-tree --no-commit-id --name-only -r -m ${GIT_COMMIT} | grep -e '\\(\\.yaml\\|\\.tpl\\)\$'")
                    if (retVal) { println "Info: No chart files modified. Skipping stage." }
                    return !retVal
                }
            }
            steps {
                container('ops-alpine') {
                    script {
                        def result = sh (returnStdout:true, script:"""
                            ROOT_DIR=\$PWD

                            # Process all modified chart templates
                            for dir in \$(git diff-tree --no-commit-id --name-only -r -m ${GIT_COMMIT} | sed -ne '/\\(\\.yaml\\|\\.tpl\\)\$/p' | sed -e 's|\\([^/]\\+/[^/]\\+\\)/.*|\\1|' | uniq); do
                                cd \$dir

                                CHART_NAME=\$(basename \$PWD)
                                CHART_VERSION=\$(cat Chart.yaml | yq e '.version' -)

                                (set +x; echo "\n[1mPackaging \$dir ...[0m")
                                if ! helm package .; then
                                    (set +x; echo "[31mError: chart could not be packaged! Review error messages.[0m")
                                    break
                                fi

                                (set +x; echo "\n[1mPublishing \$dir ...[0m")
                                if ! curl --data-binary "@\$CHART_NAME-\$CHART_VERSION.tgz" https://chartmuseum.local/api/charts -L; then
                                    (set +x; echo "[31mError: chart could not be published! Review error messages.[0m")
                                    break
                                fi

                                cd \$ROOT_DIR
                            done
                        """)

                        // Display script output
                        println "\n\u001B[1mChart package/publish stage output\u001B[0m\n$result"

                        // If issues were found, fail build
                        if (result.contains("Error:")) { currentBuild.result = "FAILURE"; }
                    }
                }
            }
        }
    }
}
