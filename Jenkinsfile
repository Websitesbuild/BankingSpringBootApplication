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

    stage('Deploy using Ansible (AWS + Azure)'){
        echo 'Deploying to AWS and Azure'

        withCredentials([
            string(credentialsId: 'azure-pass', variable: 'AZ_PASS'),
            file(credentialsId: 'aws-key', variable: 'SSH_KEY')
        ]) {

            sh """
            export ANSIBLE_HOST_KEY_CHECKING=False

            ansible-playbook -i inventory.yaml containerDeploy.yaml \
            -e httpPort=${httpPort} \
            -e containerName=${containerName} \
            -e dockerImageTag=${dockerHubUser}/${containerName}:${tag} \
            -e key_pair_path=${SSH_KEY} \
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
