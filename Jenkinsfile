pipeline {
    agent any

    stages {
        stage('Test') {
            steps {
                script {
                    def changedServices = getChangedServices()
                    echo "Changed services: ${changedServices}"
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
                        def coverage = calculateCoverage("${service}/target/jacoco.exec")
                        if (coverage < 80) {
                            error "Test coverage for ${service} is below 80%: ${coverage}%"
                        } else {
                            echo "Test coverage for ${service}: ${coverage}%"
                        }
                    }
                }
            }
        }

        stage('Build') {
            steps {
                script {
                    def changedServices = getChangedServices()
                    echo "Changed services: ${changedServices}"
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

// Hàm tính toán test coverage từ file jacoco.exec
def calculateCoverage(String execFilePath) {
    def execFile = new File(execFilePath)
    if (!execFile.exists()) {
        echo "JaCoCo exec file not found: ${execFilePath}"
        return 0
    }

    def execFileLoader = new org.jacoco.core.tools.ExecFileLoader()
    execFileLoader.load(execFile)

    def executionDataStore = execFileLoader.getExecutionDataStore()
    def coveredLines = 0
    def missedLines = 0

    // Duyệt qua các class và methods để đếm số dòng được kiểm tra (covered) và chưa kiểm tra (missed)
    executionDataStore.getContents().each { executionData ->
        executionData.getMethods().each { methodCoverage ->
            methodCoverage.getLines().each { lineCoverage ->
                if (lineCoverage.getStatus() == org.jacoco.core.analysis.LineCoverage.COVERED) {
                    coveredLines++
                } else if (lineCoverage.getStatus() == org.jacoco.core.analysis.LineCoverage.NOT_COVERED) {
                    missedLines++
                }
            }
        }
    }

    // Tính toán test coverage: C / (C + M)
    def totalLines = coveredLines + missedLines
    if (totalLines == 0) {
        return 0
    }
    
    def coverage = (coveredLines / totalLines) * 100
    return coverage
}