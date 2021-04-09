env.DOCKER_REGISTRY = 'johnojss'
env.DOCKER_IMAGE_FRONTEND = 'mern-frontend'
env.DOCKER_IMAGE_BACKEND = 'mern-backend'

pipeline {
    agent any 
    stages {
        stage('Hello World') { 
            steps {
                sh "whoami"
            }
        }
        stage('Git clone from Github') {
         steps {
             sh "git clone https://github.com/jokoss92/mern-todo-app.git"
         }   
        }
        stage('Docker Build Image') { 
            steps {
                sh "cd mern-todo-app && docker build -t $DOCKER_REGISTRY/$DOCKER_IMAGE_FRONTEND:${BUILD_NUMBER} ."
                sh "cd mern-todo-app/backend && docker build -t $DOCKER_REGISTRY/$DOCKER_IMAGE_BACKEND:${BUILD_NUMBER} ."
            }
        }
        stage('Push Docker Image') { 
            steps {
                sh "docker push $DOCKER_REGISTRY/$DOCKER_IMAGE_FRONTEND:${BUILD_NUMBER}"
                sh "docker push $DOCKER_REGISTRY/$DOCKER_IMAGE_BACKEND:${BUILD_NUMBER}"
            }
        }
        stage('Deploy Image to Kubernetes') { 
            steps {
                sh'''cd mern-todo-app && sed -i "15d" frontend.yml'''
                sh'''cd mern-todo-app && sed -i "14 a \'\\'          image: $DOCKER_REGISTRY/$DOCKER_IMAGE_FRONTEND:${BUILD_NUMBER}" frontend.yml && sed -i "s/''//" frontend.yml'''
                sh "cd mern-todo-app && kubectl apply -f frontend.yml"
                sh'''cd mern-todo-app && sed -i "15d" backend.yml'''
                sh'''cd mern-todo-app && sed -i "14 a \'\\'          image: $DOCKER_REGISTRY/$DOCKER_IMAGE_BACKEND:${BUILD_NUMBER}" backend.yml && sed -i "s/''//" backend.yml'''
                sh "cd mern-todo-app && kubectl apply -f backend.yml"                
            }                
        }
        stage('Delete Image') { 
            steps {
                sh "docker rmi $DOCKER_REGISTRY/$DOCKER_IMAGE_FRONTEND:${BUILD_NUMBER}"
                sh "docker rmi $DOCKER_REGISTRY/$DOCKER_IMAGE_BACKEND:${BUILD_NUMBER}"
            }
        }
        stage('Cleaning FIle....') { 
            steps {
                cleanWs()
            }
        }
    }
}

