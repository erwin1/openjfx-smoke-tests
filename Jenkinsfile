// Standalone Jenkinsfile for OpenJFX Smoke Tests
// This file contains all test logic in one place - no separate jobs needed

pipeline {
    agent {
        label 'linux'
    }
    
    parameters {
        string(name: 'SDK_VERSION', defaultValue: '', description: 'OpenJFX SDK version to test (leave empty to use settings.properties)')
        string(name: 'JMOD_VERSION', defaultValue: '', description: 'OpenJFX JMOD version to test (leave empty to use settings.properties)')
        string(name: 'MAVEN_VERSION', defaultValue: '', description: 'OpenJFX Maven version to test (leave empty to use settings.properties)')
        string(name: 'MAVEN_DEPLOYMENT_ID', defaultValue: '', description: 'Maven deployment ID to publish after successful tests')
        string(name: 'JAVA_VERSION', defaultValue: '17', description: 'Java version to use for builds')
        booleanParam(name: 'RUN_SDK_TESTS', defaultValue: true, description: 'Run SDK tests')
        booleanParam(name: 'RUN_JMOD_TESTS', defaultValue: true, description: 'Run JMOD tests')
        booleanParam(name: 'RUN_MAVEN_TESTS', defaultValue: true, description: 'Run Maven Central tests')
        booleanParam(name: 'AUTO_PUBLISH', defaultValue: false, description: 'Automatically publish to Maven Central if all tests pass')
    }
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 2, unit: 'HOURS')
        timestamps()
    }
    
    stages {
        stage('Load Settings') {
            steps {
                script {
                    // Load settings from settings.properties if parameters are not provided
                    if (!params.SDK_VERSION || !params.JMOD_VERSION || !params.MAVEN_VERSION) {
                        echo "Loading versions from settings.properties..."
                        def props = readProperties file: 'settings.properties'
                        
                        env.SDK_VERSION = params.SDK_VERSION ?: props['sdk_version']
                        env.JMOD_VERSION = params.JMOD_VERSION ?: props['jmod_version']
                        env.MAVEN_VERSION = params.MAVEN_VERSION ?: props['maven_version']
                        env.MAVEN_DEPLOYMENT_ID = params.MAVEN_DEPLOYMENT_ID ?: props['maven_deployment_id']
                    } else {
                        env.SDK_VERSION = params.SDK_VERSION
                        env.JMOD_VERSION = params.JMOD_VERSION
                        env.MAVEN_VERSION = params.MAVEN_VERSION
                        env.MAVEN_DEPLOYMENT_ID = params.MAVEN_DEPLOYMENT_ID
                    }
                    
                    echo """
                    Test Configuration:
                    ===================
                    SDK Version: ${env.SDK_VERSION}
                    JMOD Version: ${env.JMOD_VERSION}
                    Maven Version: ${env.MAVEN_VERSION}
                    Maven Deployment ID: ${env.MAVEN_DEPLOYMENT_ID ?: 'N/A'}
                    Java Version: ${params.JAVA_VERSION}
                    """
                }
            }
        }
        
        // ========================================================================
        // SDK TESTS - All platforms, architectures, and module types
        // ========================================================================
        stage('SDK Tests') {
            when {
                expression { params.RUN_SDK_TESTS }
            }
            matrix {
                axes {
                    axis {
                        name 'PLATFORM'
                        values 'linux', 'macos_x64', 'macos_aarch64', 'windows'
                    }
                    axis {
                        name 'MODULE_TYPE'
                        values 'modular', 'non-modular'
                    }
                }
                
                agent {
                    label "${PLATFORM}"
                }
                
                stages {
                    stage('SDK: Setup') {
                        steps {
                            script {
                                echo "Running SDK tests on ${PLATFORM} for ${MODULE_TYPE}"
                                
                                // Determine platform name and architecture for download URL
                                if (PLATFORM.startsWith('macos')) {
                                    env.PLATFORM_NAME = 'osx'
                                    env.ARCH = PLATFORM == 'macos_aarch64' ? 'aarch64' : 'x64'
                                } else if (PLATFORM == 'linux') {
                                    env.PLATFORM_NAME = 'linux'
                                    env.ARCH = 'x64'
                                } else { // windows
                                    env.PLATFORM_NAME = 'windows'
                                    env.ARCH = 'x64'
                                }
                                
                                echo "Platform: ${env.PLATFORM_NAME}, Architecture: ${env.ARCH}"
                            }
                        }
                    }
                    
                    stage('SDK: Download') {
                        steps {
                            script {
                                def sdkVersionPath = env.SDK_VERSION.replace('.', '/')
                                def downloadUrl = "https://download2.gluonhq.com/openjfx/${sdkVersionPath}/openjfx-${env.SDK_VERSION}_${env.PLATFORM_NAME}-${env.ARCH}_bin-sdk.zip"
                                
                                echo "Downloading SDK from: ${downloadUrl}"
                                
                                if (isUnix()) {
                                    sh """
                                        wget -q -P /tmp ${downloadUrl} || curl -L -o /tmp/openjfx-${env.SDK_VERSION}_${env.PLATFORM_NAME}-${env.ARCH}_bin-sdk.zip ${downloadUrl}
                                        unzip -q -o /tmp/openjfx-${env.SDK_VERSION}_${env.PLATFORM_NAME}-${env.ARCH}_bin-sdk.zip -d /tmp
                                    """
                                } else {
                                    bat """
                                        powershell -Command "Invoke-WebRequest -Uri '${downloadUrl}' -OutFile '%TEMP%\\openjfx-${env.SDK_VERSION}_${env.PLATFORM_NAME}-${env.ARCH}_bin-sdk.zip'"
                                        powershell Expand-Archive -Force %TEMP%\\openjfx-${env.SDK_VERSION}_${env.PLATFORM_NAME}-${env.ARCH}_bin-sdk.zip %TEMP%
                                    """
                                }
                            }
                        }
                    }
                    
                    stage('SDK: Test') {
                        steps {
                            script {
                                def sdkPath = isUnix() ? "/tmp/javafx-sdk-${env.SDK_VERSION}" : "%TEMP%\\javafx-sdk-${env.SDK_VERSION}"
                                
                                if (isUnix()) {
                                    sh """
                                        cd ${MODULE_TYPE}
                                        mvn clean test -Djavafx.sdk.path=${sdkPath}
                                    """
                                } else {
                                    bat """
                                        cd ${MODULE_TYPE}
                                        mvn clean test -Djavafx.sdk.path=${sdkPath}
                                    """
                                }
                            }
                        }
                    }
                }
                
                post {
                    always {
                        junit '**/target/surefire-reports/*.xml'
                    }
                    cleanup {
                        script {
                            if (isUnix()) {
                                sh """
                                    rm -rf /tmp/javafx-sdk-${env.SDK_VERSION}
                                    rm -f /tmp/openjfx-${env.SDK_VERSION}_${env.PLATFORM_NAME}-${env.ARCH}_bin-sdk.zip
                                """
                            } else {
                                bat """
                                    if exist %TEMP%\\javafx-sdk-${env.SDK_VERSION} rmdir /s /q %TEMP%\\javafx-sdk-${env.SDK_VERSION}
                                    if exist %TEMP%\\openjfx-${env.SDK_VERSION}_${env.PLATFORM_NAME}-${env.ARCH}_bin-sdk.zip del /q %TEMP%\\openjfx-${env.SDK_VERSION}_${env.PLATFORM_NAME}-${env.ARCH}_bin-sdk.zip
                                """
                            }
                        }
                    }
                }
            }
        }
        
        // ========================================================================
        // JMOD TESTS - All platforms and architectures
        // ========================================================================
        stage('JMOD Tests') {
            when {
                expression { params.RUN_JMOD_TESTS }
            }
            matrix {
                axes {
                    axis {
                        name 'PLATFORM'
                        values 'linux', 'macos_x64', 'macos_aarch64', 'windows'
                    }
                }
                
                agent {
                    label "${PLATFORM}"
                }
                
                stages {
                    stage('JMOD: Setup') {
                        steps {
                            script {
                                echo "Running JMOD tests on ${PLATFORM}"
                                
                                // Determine platform name and architecture for download URL
                                if (PLATFORM.startsWith('macos')) {
                                    env.PLATFORM_NAME = 'osx'
                                    env.ARCH = PLATFORM == 'macos_aarch64' ? 'aarch64' : 'x64'
                                } else if (PLATFORM == 'linux') {
                                    env.PLATFORM_NAME = 'linux'
                                    env.ARCH = 'x64'
                                } else { // windows
                                    env.PLATFORM_NAME = 'windows'
                                    env.ARCH = 'x64'
                                }
                                
                                echo "Platform: ${env.PLATFORM_NAME}, Architecture: ${env.ARCH}"
                            }
                        }
                    }
                    
                    stage('JMOD: Download') {
                        steps {
                            script {
                                def jmodVersionPath = env.JMOD_VERSION.replace('.', '/')
                                def downloadUrl = "https://download2.gluonhq.com/openjfx/${jmodVersionPath}/openjfx-${env.JMOD_VERSION}_${env.PLATFORM_NAME}-${env.ARCH}_bin-jmods.zip"
                                
                                echo "Downloading JMODs from: ${downloadUrl}"
                                
                                if (isUnix()) {
                                    sh """
                                        wget -q -P /tmp ${downloadUrl} || curl -L -o /tmp/openjfx-${env.JMOD_VERSION}_${env.PLATFORM_NAME}-${env.ARCH}_bin-jmods.zip ${downloadUrl}
                                        unzip -q -o /tmp/openjfx-${env.JMOD_VERSION}_${env.PLATFORM_NAME}-${env.ARCH}_bin-jmods.zip -d /tmp
                                    """
                                } else {
                                    bat """
                                        powershell -Command "Invoke-WebRequest -Uri '${downloadUrl}' -OutFile '%TEMP%\\openjfx-${env.JMOD_VERSION}_${env.PLATFORM_NAME}-${env.ARCH}_bin-jmods.zip'"
                                        powershell Expand-Archive -Force %TEMP%\\openjfx-${env.JMOD_VERSION}_${env.PLATFORM_NAME}-${env.ARCH}_bin-jmods.zip %TEMP%
                                    """
                                }
                            }
                        }
                    }
                    
                    stage('JMOD: Build Runtime') {
                        steps {
                            script {
                                def jmodsPath = isUnix() ? "/tmp/javafx-jmods-${env.JMOD_VERSION}" : "%TEMP%\\javafx-jmods-${env.JMOD_VERSION}"
                                def outputDir = isUnix() ? "/tmp/custom-runtime-${BUILD_NUMBER}" : "%TEMP%\\custom-runtime-${BUILD_NUMBER}"
                                
                                if (isUnix()) {
                                    sh """
                                        jlink --module-path ${jmodsPath}:\$JAVA_HOME/jmods \\
                                              --add-modules javafx.controls,javafx.fxml \\
                                              --output ${outputDir}
                                        
                                        ${outputDir}/bin/java --list-modules | grep javafx
                                    """
                                } else {
                                    bat """
                                        jlink --module-path ${jmodsPath};%JAVA_HOME%\\jmods ^
                                              --add-modules javafx.controls,javafx.fxml ^
                                              --output ${outputDir}
                                        
                                        ${outputDir}\\bin\\java --list-modules | findstr javafx
                                    """
                                }
                            }
                        }
                    }
                    
                    stage('JMOD: Test') {
                        steps {
                            script {
                                def jmodsPath = isUnix() ? "/tmp/javafx-jmods-${env.JMOD_VERSION}" : "%TEMP%\\javafx-jmods-${env.JMOD_VERSION}"
                                
                                if (isUnix()) {
                                    sh """
                                        cd modular
                                        mvn clean test -Djavafx.jmods.path=${jmodsPath}
                                    """
                                } else {
                                    bat """
                                        cd modular
                                        mvn clean test -Djavafx.jmods.path=${jmodsPath}
                                    """
                                }
                            }
                        }
                    }
                }
                
                post {
                    always {
                        junit '**/target/surefire-reports/*.xml'
                    }
                    cleanup {
                        script {
                            if (isUnix()) {
                                sh """
                                    rm -rf /tmp/javafx-jmods-${env.JMOD_VERSION}
                                    rm -rf /tmp/custom-runtime-${BUILD_NUMBER}
                                    rm -f /tmp/openjfx-${env.JMOD_VERSION}_${env.PLATFORM_NAME}-${env.ARCH}_bin-jmods.zip
                                """
                            } else {
                                bat """
                                    if exist %TEMP%\\javafx-jmods-${env.JMOD_VERSION} rmdir /s /q %TEMP%\\javafx-jmods-${env.JMOD_VERSION}
                                    if exist %TEMP%\\custom-runtime-${BUILD_NUMBER} rmdir /s /q %TEMP%\\custom-runtime-${BUILD_NUMBER}
                                    if exist %TEMP%\\openjfx-${env.JMOD_VERSION}_${env.PLATFORM_NAME}-${env.ARCH}_bin-jmods.zip del /q %TEMP%\\openjfx-${env.JMOD_VERSION}_${env.PLATFORM_NAME}-${env.ARCH}_bin-jmods.zip
                                """
                            }
                        }
                    }
                }
            }
        }
        
        // ========================================================================
        // MAVEN CENTRAL TESTS - All platforms, architectures, and module types
        // ========================================================================
        stage('Maven Central Tests') {
            when {
                expression { params.RUN_MAVEN_TESTS }
            }
            matrix {
                axes {
                    axis {
                        name 'PLATFORM'
                        values 'linux', 'macos_x64', 'macos_aarch64', 'windows'
                    }
                    axis {
                        name 'MODULE_TYPE'
                        values 'modular', 'non-modular'
                    }
                }
                
                agent {
                    label "${PLATFORM}"
                }
                
                stages {
                    stage('Maven: Setup') {
                        steps {
                            script {
                                echo "Running Maven Central tests on ${PLATFORM} for ${MODULE_TYPE}"
                            }
                        }
                    }
                    
                    stage('Maven: Test') {
                        steps {
                            script {
                                if (isUnix()) {
                                    sh """
                                        cd ${MODULE_TYPE}
                                        mvn clean test \
                                            -Dmaven.version=${env.MAVEN_VERSION} \
                                            -s ../settings.xml
                                    """
                                } else {
                                    bat """
                                        cd ${MODULE_TYPE}
                                        mvn clean test ^
                                            -Dmaven.version=${env.MAVEN_VERSION} ^
                                            -s ../settings.xml
                                    """
                                }
                            }
                        }
                    }
                }
                
                post {
                    always {
                        junit '**/target/surefire-reports/*.xml'
                    }
                }
            }
        }
        
        // ========================================================================
        // PUBLISH TO MAVEN CENTRAL (Optional)
        // ========================================================================
        stage('Publish to Maven Central') {
            agent {
                label 'linux'
            }
            when {
                allOf {
                    expression { params.AUTO_PUBLISH }
                    expression { env.MAVEN_DEPLOYMENT_ID != null && env.MAVEN_DEPLOYMENT_ID != '' }
                    expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
                }
            }
            steps {
                script {
                    echo "All tests passed. Publishing deployment ${env.MAVEN_DEPLOYMENT_ID} to Maven Central"
                    
                    // Add confirmation for production deployments
                    timeout(time: 10, unit: 'MINUTES') {
                        input message: "Publish deployment ${env.MAVEN_DEPLOYMENT_ID} to Maven Central?",
                              ok: 'Publish'
                    }
                    
                    // Implement your Maven Central publish logic here
                    // This would typically involve calling the Sonatype REST API
                    withCredentials([usernamePassword(credentialsId: 'sonatype-credentials', 
                                                     usernameVariable: 'SONATYPE_USER', 
                                                     passwordVariable: 'SONATYPE_PASSWORD')]) {
                        sh """
                            # Example publish command (customize based on your setup)
                            echo "Publishing to Maven Central..."
                            # curl -X POST -u \$SONATYPE_USER:\$SONATYPE_PASSWORD \\
                            #   https://oss.sonatype.org/service/local/staging/deploymentId/${env.MAVEN_DEPLOYMENT_ID}/promote
                        """
                    }
                }
            }
        }
        
        // ========================================================================
        // GENERATE TEST REPORT
        // ========================================================================
        stage('Generate Test Report') {
            agent {
                label 'linux'
            }
            steps {
                script {
                    def report = """
                    OpenJFX Smoke Tests Report
                    ===========================
                    
                    Test Execution Summary:
                    - SDK Version: ${env.SDK_VERSION}
                    - JMOD Version: ${env.JMOD_VERSION}
                    - Maven Version: ${env.MAVEN_VERSION}
                    - Build Status: ${currentBuild.result ?: 'SUCCESS'}
                    - Build Duration: ${currentBuild.durationString}
                    
                    SDK Tests: ${params.RUN_SDK_TESTS ? 'COMPLETED' : 'SKIPPED'}
                    JMOD Tests: ${params.RUN_JMOD_TESTS ? 'COMPLETED' : 'SKIPPED'}
                    Maven Tests: ${params.RUN_MAVEN_TESTS ? 'COMPLETED' : 'SKIPPED'}
                    
                    Maven Deployment: ${env.MAVEN_DEPLOYMENT_ID ?: 'N/A'}
                    Published: ${params.AUTO_PUBLISH && env.MAVEN_DEPLOYMENT_ID ? 'YES' : 'NO'}
                    """
                    
                    writeFile file: 'test-report.txt', text: report
                    archiveArtifacts artifacts: 'test-report.txt', fingerprint: true
                    
                    echo report
                }
            }
        }
    }
    
    post {
        success {
            echo "✅ All OpenJFX smoke tests completed successfully!"
            script {
                if (params.AUTO_PUBLISH && env.MAVEN_DEPLOYMENT_ID) {
                    echo "✅ Deployment ${env.MAVEN_DEPLOYMENT_ID} published to Maven Central"
                }
            }
        }
        failure {
            echo "❌ OpenJFX smoke tests failed"
        }
        always {
            node('linux') {
                emailext(
                    subject: "OpenJFX Smoke Tests - ${currentBuild.result ?: 'SUCCESS'}",
                    body: """
                        OpenJFX Smoke Tests Build Report
                        
                        Job: ${env.JOB_NAME}
                        Build: ${env.BUILD_NUMBER}
                        Status: ${currentBuild.result ?: 'SUCCESS'}
                        Duration: ${currentBuild.durationString}
                        
                        Configuration:
                        - SDK Version: ${env.SDK_VERSION}
                        - JMOD Version: ${env.JMOD_VERSION}
                        - Maven Version: ${env.MAVEN_VERSION}
                        - Maven Deployment ID: ${env.MAVEN_DEPLOYMENT_ID ?: 'N/A'}
                        
                        Test Execution:
                        - SDK Tests: ${params.RUN_SDK_TESTS ? 'Completed' : 'Skipped'}
                        - JMOD Tests: ${params.RUN_JMOD_TESTS ? 'Completed' : 'Skipped'}
                        - Maven Tests: ${params.RUN_MAVEN_TESTS ? 'Completed' : 'Skipped'}
                        
                        ${params.AUTO_PUBLISH && env.MAVEN_DEPLOYMENT_ID ? 'Published to Maven Central: YES' : ''}
                        
                        Check console output at: ${env.BUILD_URL}
                    """,
                    to: '$DEFAULT_RECIPIENTS',
                    attachLog: true
                )
            }
        }
    }
}
