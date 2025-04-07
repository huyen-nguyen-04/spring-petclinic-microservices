pipeline {
    agent any

    environment {
        CHANGED_SERVICES = ''
    }

    stages {
        stage('Determine Changed Service') {
            steps {
                script {
                    def changedServices = getChangedServices()
                    env.CHANGED_SERVICES = changedServices.join(',')
                    echo "Changed services: ${env.CHANGED_SERVICES}"
                }
            }
        }

        stage('Test') {
            steps {
                script {
                    if (env.CHANGED_SERVICES == '') {
                        echo "No services changed, skipping tests."
                        return
                    }

                    def changedServices = env.CHANGED_SERVICES.split(',').toList()
                    for (service in changedServices) {
                        echo "Testing ${service} ..."
                        sh "./mvnw clean test -f ${service}/pom.xml"
                        junit "${service}/target/surefire-reports/*.xml"
                        jacoco (
                            execPattern: "${service}/target/jacoco.exec",
                            classPattern: "${service}/target/classes",
                            sourcePattern: "${service}/src/main/java",
                            exclusionPattern: "${service}/target/test-classes"
                        )
                        echo "${service} test completed."
                    }
                }
            }
        }

        stage('Build') {
            steps {
                script {
                    if (env.CHANGED_SERVICES == '') {
                        echo "No services changed, skipping build."
                        return
                    }
                    
                    def changedServices = env.CHANGED_SERVICES.split(',').toList()
                    for (service in changedServices) {
                        echo "Building ${service} ..."
                        sh "./mvnw clean install -f ${service}/pom.xml -DskipTests"
                        echo "${service} build completed."
                    }
                }
            }
        }
    }

    post {
        success {
            setBuildStatus("Build Successful", "SUCCESS")
        }

        failure {
            setBuildStatus("Build Failed", "FAILURE")
        }
    }


}

void setBuildStatus(String message, String state) {
    step([
        $class: "GitHubCommitStatusSetter",
        reposSource: [$class: "ManuallyEnteredRepositorySource", url: "https://github.com/huyen-nguyen-04/spring-petclinic-microservices.git"],
        contextSource: [$class: "ManuallyEnteredCommitContextSource", context: "ci/jenkins/build-status"],
        errorHandlers: [[$class: "ChangingBuildStatusErrorHandler", result: "UNSTABLE"]],
        statusResultSource: [$class: "ConditionalStatusResultSource", results: [[$class: "AnyBuildResult", message: message, state: state]]]
    ]);
}

String getChangedServices() {
    def changedServices = []
    def pattern = /^spring-petclinic-.*-service$/;

    for (changeLogSet in currentBuild.changeSets) {
        for (entry in changeLogSet.getItems()) { 
            for (file in entry.getAffectedFiles()) {
                def service = file.getPath().split("/")[0]
                if (service ==~ pattern) {
                    changedServices.add(service)
                }
            }
        }
    }
    return changedServices.unique()
}