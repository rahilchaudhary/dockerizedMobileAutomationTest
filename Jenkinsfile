pipeline {
    agent {
        label 'JENKINS_NODE_NAME'
    }

    parameters {
        string(name: 'DEVICE_IP', description: 'Device IP of your Android')
        string(name: 'DOCKER_IMAGE', description: 'Docker Image Name from Docker Hub')
        string(name: 'IMAGE_TAG', defaultValue: 'latest', description: 'Image Tab from Docker Hub')
        string(name: 'application', description: 'TestNG XML File Name')

        choice(
            name: 'environment',
            choices: ['prod', 'ft'],
            description: 'Select the environment'
        )

        choice(
            name: 'suite',
            choices: ['sanity', 'smoke', 'regression'],
            description: 'Select the test type. Smoke is High Level checks. Sanity is only Critical cases. Regression is full testing.'
        )

        string(name: 'APPIUM_PORT', defaultValue: '4723', description: 'Appium Port')
        string(name: 'DOCKER_CONTAINER', defaultValue: 'app-test', description: 'Container name of your choice')


        choice(
            name: 'type',
            choices: ['ANDROID', 'API', 'WEB'],
            description: 'Platform type'
        )

        choice(
            name: 'browser',
            choices: ['native', 'api'],
            description: 'Execution mode'
        )
        string(
    name: 'TAGS',
    defaultValue: '(@Regression)',
    description: 'Cucumber filter tags'
)
    }

    environment {
        DEVICE_IP       = "${params.DEVICE_IP}"
        REPORT_FOLDER   = "${env.USERPROFILE}\\Desktop\\Reports"
        REPORTFILENAME  = "CucumberReports.html"
        ANDROID_FOLDER  = "${env.USERPROFILE}\\.android"
        DOCKER_IMAGE    = "${params.DOCKER_IMAGE}:${params.IMAGE_TAG}"
        CONTAINER_NAME  = "${params.DOCKER_CONTAINER}"
        MAVEN_PARAMS    = "-Dapplication=${params.application} -Denvironment=${params.environment} -Dsuite=${params.suite} -Dtype=${params.type} -Dbrowser=${params.browser} -Dcucumber.filter.tags='${params.TAGS}' -DappiumPort=${params.APPIUM_PORT}"
    }

    stages {

        stage('ADB Setup') {
            steps {
                echo "${env.DEVICE_IP}"
                bat '''
                adb connect %DEVICE_IP%:5555
                '''
            }
        }

        stage('Prepare Reports Folder') {
            steps {
                bat '''
                if not exist "%REPORT_FOLDER%" mkdir "%REPORT_FOLDER%"
                if exist "%REPORT_FOLDER%\\%REPORTFILENAME%.html" del "%REPORT_FOLDER%\\%REPORTFILENAME%.html"
                '''
            }
        }

        stage('Pull Image') {
            steps {
                bat 'docker pull %DOCKER_IMAGE%'
            }
        }

        stage('Start Container') {
            steps {
                bat '''
                docker stop %CONTAINER_NAME% 2>nul
                docker rm %CONTAINER_NAME% 2>nul

                docker run -d --name %CONTAINER_NAME% ^
                  -v "%ANDROID_FOLDER%:/root/.android" ^
                  -v "%REPORT_FOLDER%:/app/reports" ^
                  -e ADB_DEVICE_IP=%DEVICE_IP% ^
                  %DOCKER_IMAGE% ^
                  tail -f /dev/null
                '''
            }
        }

        stage('Run Tests') {
            steps {
                bat 'docker exec %CONTAINER_NAME% bash -c "mvn test %MAVEN_PARAMS%"'
            }
        }

        stage('Copy Report') {
            steps {
                bat '''
                docker cp %CONTAINER_NAME%:/app/target/%REPORTFILENAME%.html "%REPORT_FOLDER%\\%REPORTFILENAME%.html"
                '''
            }
        }

        stage('Verify') {
            steps {
                bat '''
                if not exist "%REPORT_FOLDER%\\%REPORTFILENAME%.html" exit /b 1
                '''
            }
        }
    }

    post {
        always {
            bat '''
            docker stop %CONTAINER_NAME% 2>nul
            docker rm %CONTAINER_NAME% 2>nul
            '''
        }
    }
}
