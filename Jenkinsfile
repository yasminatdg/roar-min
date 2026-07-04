pipeline {
    agent any
    environment {
        STAGE_VERSION = "0.0.${BUILD_NUMBER}"
        RC_VERSION = "1.0.${BUILD_NUMBER}"
        JAVA_HOME = "/opt/java/openjdk"
        PATH = "/opt/gradle-7.6/bin:${env.PATH}"
    }
    stages {
        stage('Source') {
            steps {
               deleteDir()
               checkout scm
               // sh "git clone -b main git@localhost:/git/repos/roar-min"
            }
        }
        stage('Compile') {
            steps {
                sh '''
                export JAVA_HOME=/opt/java/openjdk
                export PATH=$JAVA_HOME/bin:$PATH
                gradle -Dorg.gradle.java.home=/opt/java/openjdk -PSTAGE_VERSION=$STAGE_VERSION clean compileJava assemble
                '''              
                stash includes: '**/web*.war', name: 'roar'
            }
        }
        stage('Package-Test') {
            steps {
                unstash 'roar'
                git branch: 'test', url: 'https://github.com/brentlaster/roar-min-docker'
                sh "docker build -f Dockerfile_roar_db_image -t localhost:5000/roar-db:${STAGE_VERSION} ."
                sh "docker build -f Dockerfile_roar_web_image --build-arg warFile=web/build/libs/web-${STAGE_VERSION}*.war -t localhost:5000/roar-web:${STAGE_VERSION} . "
                sh "docker push localhost:5000/roar-db:${STAGE_VERSION}"
                sh "docker push localhost:5000/roar-web:${STAGE_VERSION}"
            }
        }
        stage('Package-Prod') {
            steps {
                unstash 'roar'
                sh "mv web/build/libs/web-${STAGE_VERSION}*.war web/build/libs/web-${RC_VERSION}*.war"
                git branch: 'main', url: 'https://github.com/brentlaster/roar-min-docker'
                sh "docker build -f Dockerfile_roar_db_image -t localhost:5000/roar-db:${RC_VERSION} ."
                sh "docker build -f Dockerfile_roar_web_image --build-arg warFile=web/build/libs/web-${RC_VERSION}*.war -t localhost:5000/roar-web:${RC_VERSION} . "
                sh "docker push localhost:5000/roar-db:${RC_VERSION}"
                sh "docker push localhost:5000/roar-web:${RC_VERSION}"
            }
        }
        stage('Deploy STAGE') {
            steps {
                git branch: 'main', url: 'git@localhost:/git/repos/roar-min-deploy'
                sh "git config --global user.email 'argocd@ci.com' && git config --global user.name 'argocd_user'"
                sh "git checkout main"
                sh "cd ./overlays/stage/db && kustomize edit set image localhost:5000/roar-db:${STAGE_VERSION}"
                sh "cd ./overlays/stage/web && kustomize edit set image localhost:5000/roar-web:${STAGE_VERSION}"
                sh "git commit -am 'Publish new staging release' && git push origin main:main || echo 'no change'"
            }
        }
        stage('Deploy RC') {
            steps {
                sh "git checkout main"
                sh "cd ./overlays/prod/db && kustomize edit set image localhost:5000/roar-db:${RC_VERSION}"
                sh "cd ./overlays/prod/web && kustomize edit set image localhost:5000/roar-web:${RC_VERSION}"
                sh "git commit -am 'Publish new release candidate' && git push origin main:main || echo 'no change'"
            }
        }
    }
}
