podTemplate(
    inheritFrom: "maven", 
    label: "myJenkins", 
    cloud: "openshift", 
    volumes: [
        persistentVolumeClaim(claimName: "m2repo", mountPath: "/tmp/.m2")
    ]) {

    node("myJenkins") {
    
        stage("Git Checkout") {
            echo "Checking out git repository"
            checkout scm
        }

        stage("Build") {
            echo 'Building project'
            sh "mvn package"
        }

        stage("DEV - Image build") {
            echo 'Building docker image and deploying to Dev'
            sh "oc new-project 12factor-dev || echo 'Project exists'"
            sh "oc policy add-role-to-user admin developer -n 12factor-dev"
            sh "oc project 12factor-dev"
            sh "oc new-build -n 12factor-dev --binary --name=my12factorapp || echo 'Build exists'"
            sh "oc start-build my12factorapp -n 12factor-dev --from-dir=. --follow"
            echo "This is the build number: ${env.BUILD_NUMBER}"
        }
        
        stage ("DEV - Run"){
            sh "oc new-app my12factorapp -n 12factor-dev  || echo 'Aplication already Exists'"
            sh "oc expose service my12factorapp -n 12factor-dev || echo 'Service already exposed'"
            sh "oc set probe dc/my12factorapp -n 12factor-dev --readiness --get-url=http://:8080/api/health"
        }

        stage("Automated tests") {
            echo 'This stage simulates automated tests'
            sh "mvn -B -Dmaven.test.failure.ignore verify"
        }

        stage("QA - App Deploy") {
            echo 'Deploying to QA'
            sh "oc new-project 12factor-qa || echo 'Project exists'"
            sh "oc policy add-role-to-user admin developer -n 12factor-qa"
            sh "oc project 12factor-qa"
            sh "oc policy add-role-to-user system:image-puller system:serviceaccount:12factor-qa:default -n 12factor-dev"
            sh "oc tag 12factor-dev/my12factorapp:latest 12factor-qa/my12factorapp:latest"
            sh "oc new-app my12factorapp -n 12factor-qa  || echo 'Aplication already Exists'"
            sh "oc expose service my12factorapp -n 12factor-qa || echo 'Service already exposed'"
            sh "oc set probe dc/my12factorapp -n 12factor-qa  --readiness --get-url=http://:8080/api/health"
        }

        stage("Approve to PROD") {
            input 'Approve to production?'
        }

        stage("PROD - App Deploy") {
            echo 'Deploying to production'
            sh "oc new-project 12factor || echo 'Project exists'"
            sh "oc policy add-role-to-user admin developer -n 12factor"
            sh "oc project 12factor"
            sh "oc policy add-role-to-user system:image-puller system:serviceaccount:12factor:default -n 12factor-qa"
            sh "oc tag 12factor-qa/my12factorapp:latest 12factor/my12factorapp:${env.BUILD_NUMBER}"
            sh "oc new-app -n 12factor --name my12factorapp-${env.BUILD_NUMBER} my12factorapp:${env.BUILD_NUMBER}" 
        }

        stage ("PROD - Canary deploy") {
            sh "oc new-project 12factor || echo 'Project exists'"
            sh "oc project 12factor"
            sh "oc set probe dc/my12factorapp -n 12factor --readiness --get-url=http://:8080/api/health"
            sh "oc patch route/my12factorapp -n 12factor  -p '{\"spec\": {\"to\": {\"name\": \"demo\", \"weight\": 50 }}}'"
            sh "oc patch route demo -n 12factor  -p '{\"spec\":{\"alternateBackends\":[{\"kind\":\"Service\", \"name\":\"my12factorapp-${env.BUILD_NUMBER}\", \"weight\": 50}]}}'"
        }
    }
}    
