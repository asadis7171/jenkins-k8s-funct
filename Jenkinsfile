def call(body) {
    def config = [:]
    body.resolveStrategy = Closure.DELEGATE_FIRST
    body.delegate = config
    body()
}

pipeline {
    agent any

    parameters {
        choice(name: 'account', choices: ['dev', 'qa', 'stage', 'prod'], description: 'Select the environment.')
        string(name: 'commit_id', defaultValue: 'latest', description: 'provide commit id.')
    }

    environment {
        COMMITID            = "${GIT_COMMIT}"
        dev_dh_creds        = 'docker-cred'
        dev_registry        = 'asadis7171/dev'
        dev_registry_endpoint = 'https://' + "${env.registryURI}" + "${env.dev_registry}"
        dev_image           = "${env.dev_registry}" + ":" + "${env.COMMITID}"
        prod_dh_creds       = 'dh_cred_prod'
        prod_image          = "${env.registryURI}" + "${env.prod_registry}" + ':' + "${env.COMMITID}"
        prod_registry       = 'asadis7171/prod'
        prod_registry_endpoint = 'https://' + "${env.registryURI}" + "${env.prod_registry}"
        qa_dh_creds         = 'dh_cred_qa'
        qa_image            = "${env.registryURI}" + "${env.qa_registry}" + ':' + "${env.COMMITID}"
        qa_registry         = 'asadis7171/qa'
        qa_registry_endpoint = 'https://' + "${env.registryURI}" + "${env.qa_registry}"
        registryURI         = 'registry.hub.docker.com/'
        stage_dh_creds      = 'dh_cred_stage'
        stage_image         = "${env.registryURI}" + "${env.stage_registry}" + ':' + "${env.COMMITID}"
        stage_registry      = 'asadis7171/stage'
        stage_registry_endpoint = 'https://' + "${env.registryURI}" + "${env.stage_registry}"
        DEV_CONFIG      =       "dev_kube_config"
        QA_CONFIG       =       "qa_kube_config"
        STAGE_CONFIG    =       "stage_kube_config"
        PROD_CONFIG     =       "prod_kube_config"
    }

    stages {

        stage('Docker Image Build IN Dev') {
            when {
                expression {
                    params.account == 'dev'
                }
            }

            steps {
                echo "Building Docker Image Logging in to Docker Hub & Pushing the Image"
                dockerBuildPush(env.dev_registry_endpoint , env.dev_dh_creds)

                sh 'echo Image Pushed to DEV'
            }
        }

        stage('Pull Tag push to QA') {
            when {
                expression {
                    params.account == 'qa'
                }
            }

            steps {
                dockerPullTagPush(env.dev_registry_endpoint , env.dev_dh_creds , env.dev_image , env.qa_registry_endpoint , env.qa_dh_creds , env.qa_image)
            }
        }

        stage('Pull Tag push to stage') {
            when {
                expression {
                    params.account == 'stage'
                }
            }

            steps {
                dockerPullTagPush(env.qa_registry_endpoint, env.qa_dh_creds, env.qa_image, env.stage_registry_endpoint, env.stage_dh_creds, env.stage_image)
            }
        }

        stage('Pull Tag push to Prod') {
            when {
                expression {
                    params.account == 'prod'
                }
            }

            steps {
                dockerPullTagPush(env.env.stage_registry_endpoint, env.stage_dh_creds, env.stage_image , env.prod_registry_endpoint, env.prod_dh_creds, env.prod_image)
            }
        }

        stage('DEPLOY TO K8S DEV') {
            when {
                expression {
                    params.account == 'dev'
                }
            }
            steps {
                deployOnK8s(env.DEV_CONFIG , env.ACCOUNT , env.COMMITID)
            }
        }

        stage('DEPLOY TO K8S QA') {
            when {
                expression {
                    params.account == 'qa'
                }
            }
            steps {
                deployOnK8s(env.QA_CONFIG , env.ACCOUNT , env.COMMITID)
            }
        }

        stage('DEPLOY TO K8S STAGE') {
            when {
                expression {
                    params.account == 'stage'
                }
            }
            steps {
                deployOnK8s(env.STAGE_CONFIG , env.ACCOUNT , env.COMMITID)
            }
        }

        stage('DEPLOY TO K8S PROD') {
            when {
                expression {
                    params.account == 'prod'
                }
            }
            steps {
                deployOnK8s(env.PROD_CONFIG , env.ACCOUNT , env.COMMITID)
            }
        }
    }

    post {
        always {
            echo 'Deleting Workspace from shared Lib'
            deleteDir() /* clean up our workspace */
        }
    }
}

def dockerBuildPush( String SRC_DH_URL , String SRC_DH_CREDS , String SRC_DH_TAG ) {
    def app = docker.build(SRC_DH_TAG)
    docker.withRegistry("https://" + SRC_DH_URL , SRC_DH_CREDS) {
        app.push()
    }
}

def dockerPullTagPush( String SRC_DH_URL , String SRC_DH_CREDS , String SRC_DH_TAG , String DEST_DH_URL , String DEST_DH_CREDS , String DEST_DH_TAG ) {
    docker.withRegistry("https://" + SRC_DH_URL , SRC_DH_CREDS) {
        docker.image(SRC_DH_TAG).pull()
    }
    sh 'echo Image pulled successfully...'

    sh 'echo Taggig Docker image...'
    sh "docker tag ${SRC_DH_TAG} ${DEST_DH_TAG}" 

    docker.withRegistry("https://" + DEST_DH_URL , DEST_DH_CREDS) {
        docker.image(DEST_DH_TAG).push()
    }
   
    sh 'echo Image Pushed successfully...'
    sh 'echo Deleting Local docker Images'
    sh "docker image rm ${SRC_DH_TAG}"  
    sh "docker image rm ${DEST_DH_TAG}" 
}

def deployOnK8s(String KUBE_CONFIG, String ACCOUNT, String COMMIT) {
    withKubeConfig(credentialsId: "${KUBE_CONFIG}", restrictKubeConfigAccess: true) {
        sh 'echo Deploying application on ${ACCOUNT} K8S cluster'
        sh 'echo Replacing K8S manifests files with sed....'
        sh "sed -i -e 's/{{ACCOUNT}}/${ACCOUNT}/g' -e 's/{{COMMITID}}/${COMMIT}/g' KUBE/*"
        sh 'echo K8S manifests files after replace with sed ...'
        sh 'cat KUBE/deployment.yaml'
        sh 'kubectl apply -f KUBE/.'
    }
}
