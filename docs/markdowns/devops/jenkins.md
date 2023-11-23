---
icon: fontawesome/brands/jenkins
---

### Jenkins Pipeline传大文件

通过git-lfs可以对大文件进行管理，利用该特性使用jenkins pipeline将本地build出的artifacts同步进数据中心中

以下Jenkinsfile为示例

=== "Jenkinsfile"
    ```groovy
    pipeline {
        agent {
            label 'master'   
        }

        stages {
            stage('Checkout from DXC GitHub') {
                steps {
                    checkout scm
                }
            }
            stage('Start packages sync') {
                steps {
                    script {
                        ansiColor('xterm') {
                            echo sh(script: '''find . -type f -regex ".*\\.\\(tgz\\|tar.gz\\|jar\\|zip\\)"|xargs sha256sum -t > packages.tmp''', returnStdout: true)
                            echo "\u001B[34m ****** Start latest SHA256SUM list ******"
                            package_tmp = sh(script: '''cat packages.tmp''', returnStdout: true).trim()
                            echo "\u001B[34m${package_tmp}"
                            echo "\u001B[34m ****** End latest SHA256SUM list ******"
                            echo "****** Start old SHA256SUM list"
                            echo sh(script: ''' cat packages.conf''', returnStdout: true)
                            echo "****** End old SHA256SUM list"
                            
                            echo "\u2713 Start sync below packages....."
                            packagesToBeSync = sh(script: '''grep -vwf packages.conf packages.tmp || true''', returnStdout: true).trim()
                            echo "\u001B[34m${packagesToBeSync} "

                            echo "Syncing, please wait ..."
                            echo sh(script: '''#!/usr/bin/env bash
                                    pakcage_number=`grep -vwf packages.conf packages.tmp |awk 'BEGIN {FS=" "} {print $1, $2}'|wc -l`
                                    if [ $pakcage_number -ne 0 ] 
                                    then
                                        for line in `grep -vwf packages.conf packages.tmp |awk 'BEGIN {FS=" "} {print $2}'`
                                        do
                                            scp -q $line root@n2.pln-n2-depot2:/repo/www/deploy/ngp-packages/$line
                                        done
                                        cat packages.tmp > packages.conf 
                                        rm -rf packages.tmp
                                        git commit -am "updated packages.conf after sync"
                                        git push origin HEAD:master
                                    else
                                        echo "No new packages need to be sync!"
                                    fi
                                    ''', returnStdout: true)

                        }
                    }
                }
            }
            stage('Define variables') {
                steps {
                    script {
                        propel_ha_params = []
                    }
                }
            }
            stage('Get hash values') {
                when {
                    expression { packagesToBeSync }
                }
                steps {
                    script {
                        echo "build the jenkins project parameters from the packages:\n${packagesToBeSync}"
                        def paramsToBeBuilt = packagesToBeSync.readLines()
                        def propel_ha_mapping = sh(script: 'cat propel-ha-parameters-mapping.csv', returnStdout: true).trim().readLines()
                        paramsToBeBuilt.each {
                            def (hash, pkg) = it.tokenize(' ')
                            def found = propel_ha_mapping.find { it.contains(pkg) }
                            if (found) {
                                def (_, param) = found.tokenize(',')
                                propel_ha_params << "${param}=${hash}"
                                return
                            }                            
                        }
                    }
                }
            }
            stage('Deploy NGP Packages') {
                when {
                    expression { propel_ha_params }
                }
                steps {
                    script {
                        params = propel_ha_params.join('&')
                        echo "parameters and hashes: ${params}"
                        sh """
                            curl -s -k -u li:xxx -X POST "https://xxx/jenkins/job/(Self-Service)Deploy%20NGP%20Package%20to%20FT1(Pipeline)/buildWithParameters?delay=0sec&${params}"
                        """
                    }
                }
            }
        }
        post {
            failure {
                emailext mimeType: 'text/html',
                        from: 'xxx@dxc.com',
                        to: 'ngp_mpc_adapter_dev@dxc.com',
                        replyTo: '$DEFAULT_REPLYTO',
                        subject: '$DEFAULT_SUBJECT',
                        body: '${SCRIPT, template="chef.template"}'
                error 'something running failure, check the log'
            }
            success {
                emailext mimeType: 'text/html',
                        from: 'xxx@dxc.com',
                        to: '$DEFAULT_RECIPIENTS',
                        replyTo: '$DEFAULT_REPLYTO',
                        subject: '$DEFAULT_SUBJECT',
                        body: '${SCRIPT, template="chef.template"}'
            }
        }
    }
    ```

### Jenkins Pipeline 完整的安全检查示例
=== "Jenkinsfile"
    ```groovy
    #!/usr/bin/env groovy
    pipeline {
        options {
            timeout(time: 1, unit: 'HOURS')
            timestamps()
            disableConcurrentBuilds()
            buildDiscarder(logRotator(numToKeepStr: '5', artifactNumToKeepStr: '5', daysToKeepStr: '5'))
        }

        agent any

        environment {
            TEAM_SHORTNAME = "vpc-pat" 
            APPLICATION_NAME = "vpc-biz-aggregator"
            DOCKER_REGISTRY = "docker-registry-local.com"
            DOCKER_REGISTRY_URL = 'https://docker-registry-local.com'
            DOCKER_REGISTRY_REPO = 'vpc-compute-docker'
            DEPLOYMENT_ENV = 'dev' //ft1, prod
            SPRING_PROFILES_ACTIVE = "dev" //ft1, prod
            SONARHOST = "https://xxx"
            SONARTOKEN = credentials("vpc-sonarqube-token")
            ARTIFACT = readMavenPom().getArtifactId()
            VERSION = readMavenPom().getVersion()
        }

        stages {
            stage('Cleaning') {
                agent {
                    docker {
                        image "docker-registry-local.com/vpc-compute-docker/devops/maven:3.6-openjdk-11-slim"
                        reuseNode true
                        args '-v $HOME/.m2:/root/.m2 -v $WORKSPACE:/workspace" -u="root" -w /workspace'
                        registryUrl "https://docker-registry-local.com"
                        registryCredentialsId "vpc-docker-registry-credential"
                    }
                }
                steps {
                    checkout scm
                    script {
                        notifyBuild()
                        sh "echo Branch name is : ${env.BRANCH_NAME}"
                        sh "echo ${env.GIT_COMMIT}"
                        sh "echo Deployment target environment is : ${DEPLOYMENT_ENV}"
                        sh "echo Active profile is : ${SPRING_PROFILES_ACTIVE}"
                        sh "echo Current build id is : ${BUILD_ID}"
                        sh "echo Image target URI: ${DOCKER_REGISTRY_URL}/${DOCKER_REGISTRY_REPO}/${TEAM_SHORTNAME}/${APPLICATION_NAME}:${VERSION}"
                        sh "echo IMAGE: ${ARTIFACT}"
                        sh "echo VERSION: ${VERSION}"
                        sh "mvn -B -DskipTests clean package"
                        archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
                    }
                }
            }
            stage('Unit Test') {
                agent {
                    docker {
                        image "docker-registry-local.com/vpc-compute-docker/devops/maven:3.6-openjdk-11-slim"
                        reuseNode true
                        args '-v $HOME/.m2:/root/.m2 -v $WORKSPACE:/workspace" -u="root" -w /workspace'
                        registryUrl "https://docker-registry-local.com"
                        registryCredentialsId "vpc-docker-registry-credential"
                    }
                }
                steps {
                    echo "Starting UT ..."
                    sh 'mvn test'
                    sh 'mvn -DskipTests surefire-report:report'
                }
                post {
                    always {
                    sh 'echo save unit test results'
                    junit '**/target/surefire-reports/TEST-*.xml'
                    }
                }
            }
            stage('Sonar Analysis') {
                agent {
                    docker {
                        image "docker-registry-local.com/vpc-compute-docker/devops/maven:3.6-openjdk-11-slim"
                        reuseNode true
                        args '-v $HOME/.m2:/root/.m2 -v $WORKSPACE:/workspace" -u="root" -w /workspace'
                        registryUrl "https://docker-registry-local.com"
                        registryCredentialsId "vpc-docker-registry-credential"
                    }
                }
                steps {
                    echo "Start Sonar Scanner ..."
                    sh "mvn jacoco:report sonar:sonar -Dsonar.host.url=$SONARHOST -Dsonar.login=$SONARTOKEN"
                }
            }
            stage('Sonar Quality Gate') {
                steps {
                sh 'bash -x gate.sh $SONARHOST $SONARTOKEN'
                }
            }
    //         stage('Dependency check') {
    //             agent {
    //                 docker {
    //                     image "docker-registry-local.com/vpc-compute-docker/devops/maven:3.6-openjdk-11-slim"
    //                     reuseNode true
    //                     args '-v $HOME/.m2:/root/.m2 -v $WORKSPACE:/workspace" -u="root" -w /workspace'
    //                     registryUrl "https://docker-registry-local.com"
    //                     registryCredentialsId "vpc-docker-registry-credential"
    //                 }
    //             }
    //             steps {
    //                 sh "mvn --batch-mode dependency-check:check"
    //             }
    //             post {
    //             always {
    //                 publishHTML(target:[
    //                     allowMissing: true,
    //                     alwaysLinkToLastBuild: true,
    //                     keepAll: true,
    //                     reportDir: 'target',
    //                     reportFiles: 'dependency-check-report.html',
    //                     reportName: "OWASP Dependency Check Report"
    //                 ])
    //                 }
    //             }
    //         }
            stage('Dockerize and Image Scan') {
                when {
                    branch 'develop'
                }
                environment {
                    DEPLOYMENT_ENV = 'dev'
                    SPRING_PROFILES_ACTIVE = 'dev'
                }
                steps {
                    script{
                        docker.withRegistry("${DOCKER_REGISTRY_URL}", 'vpc-docker-registry-credential') {
                            def image = docker.build("${DOCKER_REGISTRY_REPO}/${TEAM_SHORTNAME}/${APPLICATION_NAME}:${VERSION}-${DEPLOYMENT_ENV}", "--build-arg SPRING_PROFILES_ACTIVE=${SPRING_PROFILES_ACTIVE} .")
                            sh '''
                                docker stop db && docker rm -f db && docker stop clair && docker rm -f clair
                                docker run -d --name db arminc/clair-db
                                sleep 15
                                docker run -p 6060-6061:6060-6061 --link db:postgres -d --name clair arminc/clair-local-scan
                                sleep 3
                                wget -qO clair-scanner https://github.com/arminc/clair-scanner/releases/download/v12/clair-scanner_linux_amd64 && chmod +x clair-scanner
                                ./clair-scanner --ip="172.17.0.1" --threshold="Critical" --report="clair-result.json" ${DOCKER_REGISTRY_REPO}/${TEAM_SHORTNAME}/${APPLICATION_NAME}:${VERSION}-${DEPLOYMENT_ENV}
                                wget -qO clairctl https://github.com/jgsqware/clairctl/releases/download/v1.2.8/clairctl-linux-amd64 && chmod +x clairctl
                                ./clairctl --config=clairctl.yml report --local ${DOCKER_REGISTRY_REPO}/${TEAM_SHORTNAME}/${APPLICATION_NAME}:${VERSION}-${DEPLOYMENT_ENV}
                            '''
                            image.push()
                            sh (script: "docker inspect -f '{{ .Id }}' ${image.id} | xargs docker rmi -f", returnStatus: true)
                        }
                        archiveArtifacts artifacts: 'clair-result.json', fingerprint: true
                        archiveArtifacts artifacts: '**/html/*.html', fingerprint: true
                    }
                }
            }
            stage('Promote Image for FT2') {
                when {
                    branch 'develop'
                }
                environment {
                    DEPLOYMENT_ENV = 'ft2'
                    SPRING_PROFILES_ACTIVE = 'ft2'
                }
                steps{
                    script{
                        docker.withRegistry("${DOCKER_REGISTRY_URL}", 'vpc-docker-registry-credential') {
                            def theImage = docker.image("${DOCKER_REGISTRY}/${DOCKER_REGISTRY_REPO}/${TEAM_SHORTNAME}/${APPLICATION_NAME}:${VERSION}-dev")
                            theImage.pull()
                            theImage.push("${VERSION}-${DEPLOYMENT_ENV}")
                        }
                    }
                }
            }
            stage('QA Approval') {
                when {
                    branch 'master'
                }
                steps {
                    script {
                        def doPromote = false
                        timeout(time: 1, unit: 'HOURS') {
                            doPromote = input id: 'Promote', message: 'Is it ready for QA?'
                            notifyBuild('SUSPENDED', 'Can any QA approve this?')
                        }
                    }
                }
            }
            stage('Promote Image for QA') {
                when {
                    branch 'develop'
                }
                environment {
                    DEPLOYMENT_ENV = 'env1'
                    SPRING_PROFILES_ACTIVE = 'env1'
                }
                steps{
                    script{
                        docker.withRegistry("${DOCKER_REGISTRY_URL}", 'vpc-docker-registry-credential') {
                            def theImage = docker.image("${DOCKER_REGISTRY}/${DOCKER_REGISTRY_REPO}/${TEAM_SHORTNAME}/${APPLICATION_NAME}:${VERSION}-dev")
                            theImage.pull()
                            theImage.push("${VERSION}-${DEPLOYMENT_ENV}")
                        }
                    }
                }
            }
            stage('Promote Image for newpro') {
                when {
                    branch 'develop'
                }
                environment {
                    DEPLOYMENT_ENV = 'newpro'
                    SPRING_PROFILES_ACTIVE = 'redhat8'
                }
                steps{
                    script{
                        docker.withRegistry("${DOCKER_REGISTRY_URL}", 'vpc-docker-registry-credential') {
                            def theImage = docker.image("${DOCKER_REGISTRY}/${DOCKER_REGISTRY_REPO}/${TEAM_SHORTNAME}/${APPLICATION_NAME}:${VERSION}-dev")
                            theImage.pull()
                            theImage.push("${VERSION}-${DEPLOYMENT_ENV}")
                        }
                    }
                }
            }
            stage('Security Review and PCR Approval') {
                when {
                    branch pattern: '^v*-RELEASE', comparator: "REGEXP"
                }
                steps {
                    script {
                        def doPromote = false
                        timeout(time: 1, unit: 'HOURS') {
                            doPromote = input id: 'Promote', message: 'Is it ready for MTP?'
                            notifyBuild('SUSPENDED', 'Can you approve this?')
                        }
                    }
                }
            }
            stage('Promote Image for MTP') {
                when {
                    branch pattern: '^v*-RELEASE', comparator: "REGEXP"
                }
                environment {
                    DEPLOYMENT_ENV = 'prod'
                    SPRING_PROFILES_ACTIVE = 'prod'
                }
                steps{
                    script{
                        docker.withRegistry("${DOCKER_REGISTRY_URL}", 'vpc-docker-registry-credential') {
                            def theImage = docker.image("${DOCKER_REGISTRY}/${DOCKER_REGISTRY_REPO}/${TEAM_SHORTNAME}/${APPLICATION_NAME}:${VERSION}-env1")
                            theImage.pull()
                            theImage.push("${VERSION}-${DEPLOYMENT_ENV}")
                            sh "docker logout docker-registry-local.com"
                        }
                    }
                }
            }
        }

        post {
            success {
                notifyBuild('SUCCESS')
            }
            unsuccessful {
                notifyBuild('FAILED')
            }
        }
    }

    def notifyBuild(String buildStatus = 'STARTED', String customMessage = '') {
    // build status of null means successful
        buildStatus = buildStatus ?: 'SUCCESS'

        // Default values
        def colorName = 'RED'
        def colorCode = '#FF0000'
        def subject = "${buildStatus}: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'"
        def summary = "${subject}"
        if (customMessage != ''){
            summary = customMessage
        }
        def details = """<p>STARTED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]':</p>
        <p>Check console output at &QUOT;<a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}]</a>&QUOT;</p>"""

        // Override default values based on build status
        if (buildStatus == 'STARTED') {
            color = 'BLUE'
            colorCode = '#0000FF'
        } else if (buildStatus == 'SUCCESS') {
            color = 'GREEN'
            colorCode = '#00FF00'
        } else if (buildStatus == 'SUSPENDED') {
            color = 'YELLOW'
            colorCode = '#FFFF00'
        } else {
            color = 'RED'
            colorCode = '#FF0000'
        }

        office365ConnectorSend(color: colorCode,
                            message: summary,
                            status: buildStatus,
                            factDefinitions: [
                                                [name: "Commit #:", template: "[${env.GIT_COMMIT.take(7)}](${env.GIT_HTTP_URL}/commit/${env.GIT_COMMIT})"]
                                            ],
                            webhookUrl:'https://outlook.office.com/webhook/xxx'
        )
    }
    ```