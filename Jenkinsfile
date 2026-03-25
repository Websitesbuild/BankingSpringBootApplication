node {

    def tag="", dockerHubUser="", containerName="", httpPort=""

    stage('Prepare Environment'){
        echo 'Initialize Environment'

        tag = "3.0"
        containerName = "bankingapp"
        httpPort = "8989"

        withCredentials([usernamePassword(credentialsId: 'dockerHubAccount', usernameVariable: 'dockerUser', passwordVariable: 'dockerPassword')]) {
            dockerHubUser = dockerUser
        }
    }

    stage('Code Checkout'){
        try {
            checkout scm
        } catch(Exception e){
            echo 'Git Checkout Failed'
            currentBuild.result = "FAILURE"
            error("Stopping pipeline")
        }
    }

    stage('Maven Build'){
        sh "mvn clean package"
    }

    stage('Docker Build'){
        echo 'Building Docker Image'
        sh "docker build -t ${dockerHubUser}/${containerName}:${tag} --pull --no-cache ."
    }

    stage('Docker Login & Push'){
        echo 'Pushing Image to DockerHub'

        withCredentials([usernamePassword(credentialsId: 'dockerHubAccount', usernameVariable: 'dockerUser', passwordVariable: 'dockerPassword')]) {
            sh """
            echo ${dockerPassword} | docker login -u ${dockerUser} --password-stdin
            docker push ${dockerUser}/${containerName}:${tag}
            """
        }
    }

    // 🔥 AWS Deployment (SSH Key)
    stage('Deploy AWS'){
        echo 'Deploying to AWS'

        sh """
        export ANSIBLE_HOST_KEY_CHECKING=False

        ansible-playbook -i inventory.yaml containerDeploy.yaml \
        --limit aws \
        -e httpPort=${httpPort} \
        -e containerName=${containerName} \
        -e dockerImageTag=${dockerHubUser}/${containerName}:${tag} \
        -e key_pair_path=/var/lib/jenkins/server.pem \
        --become
        """
    }

    // 🔐 Azure Deployment (Password)
    stage('Deploy Azure'){
        echo 'Deploying to Azure'

        withCredentials([string(credentialsId: 'azure-pass', variable: 'AZ_PASS')]) {
            sh """
            export ANSIBLE_HOST_KEY_CHECKING=False

            ansible-playbook -i inventory.yaml containerDeploy.yaml \
            --limit azure \
            -e httpPort=${httpPort} \
            -e containerName=${containerName} \
            -e dockerImageTag=${dockerHubUser}/${containerName}:${tag} \
            -e ansible_password=${AZ_PASS} \
            --become
            """
        }
    }

    stage('Verify Deployment'){
        echo 'Checking running containers'

        sh """
        ansible all -i inventory.yaml -a "docker ps"
        """
    }

    stage('Cleanup'){
        echo 'Cleaning unused Docker images'
        sh "docker system prune -f || true"
    }

    echo "✅ Pipeline Completed Successfully"
}
