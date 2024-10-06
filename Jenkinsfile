node {
    properties([disableConcurrentBuilds()])
    def version
    def branch_name
    def version_suffix

    stage('Preparation') {
        checkout scm
        branch_name = env.BRANCH_NAME

        if (branch_name == "master") {
            version_suffix = ""
        }
        if (branch_name.startsWith("release/")) {
            version_suffix = "rc"
        }
        
        version = sh returnStdout: true, script: 'head -n 1 version.txt'
        version = version+version_suffix
    }
    stage('Build') {
        parallel(
            frontend: {
                if (branch_name == "master") {
                    sh 'docker build --network host -f docker/builder/frontend/Dockerfile -t forma-fe-builder --build-arg VITE_APP_ENV=production .'
                }
                if (branch_name.startsWith("release/")){
                    sh 'docker build --network host -f docker/builder/frontend/Dockerfile -t forma-fe-builder --build-arg VITE_APP_ENV=qa .'
                }
                sh 'docker run --rm -v $(pwd)/frontend/dist/:/usr/src/app/dist forma-fe-builder'
            },
            backend: {
                sh 'docker build -f docker/builder/backend/Dockerfile -t forma-be-builder .'
                sh 'docker run --rm -v $(pwd)/backend/dist/:/usr/src/app/dist/app/src forma-be-builder'
            }
        )
    }
    stage('Build Container') {
        sh "docker build -f docker/production/frontend/Dockerfile -t peworegistry.azurecr.io/forma_frontend ."
        sh "docker build -f docker/production/backend/Dockerfile -t peworegistry.azurecr.io/forma_backend ."
        sh "docker build -f docker/production/proxy/Dockerfile -t peworegistry.azurecr.io/forma_proxy ."
    }
    stage('Tag Container') {
        sh "docker tag peworegistry.azurecr.io/forma_frontend peworegistry.azurecr.io/forma_frontend:"+version
        sh "docker tag peworegistry.azurecr.io/forma_backend peworegistry.azurecr.io/forma_backend:"+version
        sh "docker tag peworegistry.azurecr.io/forma_proxy peworegistry.azurecr.io/forma_proxy:"+version
    }
    stage('Push Container') {
        sh "docker push peworegistry.azurecr.io/forma_frontend:latest"
        sh "docker push peworegistry.azurecr.io/forma_frontend:"+version
        sh "docker push peworegistry.azurecr.io/forma_backend:latest"
        sh "docker push peworegistry.azurecr.io/forma_backend:"+version
        sh "docker push peworegistry.azurecr.io/forma_proxy:latest"
        sh "docker push peworegistry.azurecr.io/forma_proxy:"+version
    }
    stage('Deploy') {
        if (branch_name == "master") {
            sh 'export VERSION='+version+' && DOCKER_HOST=10.113.21.158:2375 docker-compose -f docker-compose-prod.yml pull'
            sh 'export VERSION='+version+' && DOCKER_HOST=10.113.21.158:2375 docker-compose -f docker-compose-prod.yml up -d --remove-orphans'
        }
        if (branch_name.startsWith("release/")) {
            sh 'export VERSION=latest && DOCKER_HOST=10.113.21.167:2375 docker-compose -f docker-compose-prod-qa.yml pull'
            sh 'export VERSION=latest && DOCKER_HOST=10.113.21.167:2375 docker-compose -f docker-compose-prod-qa.yml up -d --remove-orphans'
        }
    }
}
