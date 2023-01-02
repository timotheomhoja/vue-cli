# Vue CLI [![Build Status](https://circleci.com/gh/vuejs/vue-cli/tree/dev.svg?style=shield)](https://circleci.com/gh/vuejs/vue-cli/tree/dev) [![Windows Build status](https://ci.appveyor.com/api/projects/status/rkpafdpdwie9lqx0/branch/dev?svg=true)](https://ci.appveyor.com/project/yyx990803/vue-cli/branch/dev) [![lerna](https://img.shields.io/badge/maintained%20with-lerna-cc00ff.svg)](https://lerna.js.org/)

## Status

Vue CLI is now in maintenance mode. For new Vue 3 projects, please use [create-vue](https://github.com/vuejs/create-vue) to scaffold [Vite](https://vitejs.dev/)-based projects.

## Documentation

Docs are available at https://cli.vuejs.org/ - we are still working on refining it and contributions are welcome!

## Contributing

Please see [contributing guide](https://github.com/vuejs/vue-cli/blob/dev/.github/CONTRIBUTING.md).

## License

[MIT](https://github.com/vuejs/vue-cli/blob/dev/LICENSE)

#jenkins for vue js
pipeline {

    agent any

    options {
        ansiColor('xterm')
    }

    environment {
        IMG = "baridibaridi_app:${BUILD_ID}"
        IMG_TAGGED = "baidibaridi_app:latest"
        TG_HOST = "ssh://ec2-user@ip"
        TG_SSH  = "ip"
        TG_USR  = "ec2-user"
        TG_PORT = 9000
    }

    stages {
        // Build the image
        stage("Build") {
            steps {
                sh "docker build -t ${IMG} ."
                sh "docker tag ${IMG} ${IMG_TAGGED}"
            }
        }

        // Deliver the image to the tgt server
        stage("Deliver") {
            steps {
                sshagent(credentials: ['baridibaridi_app']) {
                    sh'''
                        [ -d ~/.ssh ] || mkdir ~/.ssh
                        ssh-keyscan -t rsa,dsa ${TG_SSH} >> ~/.ssh/known_hosts
                        docker save ${IMG_TAGGED} | ssh -C ${TG_USR}@${TG_SSH} docker load
                    '''
                }
            }
        }


        stage("Deploy-Prod") {

            steps {
                // The use of single-quotes instead of double-quotes to define the script
                // (the implicit parameter to sh) in Groovy.
                // The single-quotes will cause the secret to be expanded by the shell
                // as an environment variable. The double-quotes are potentially less
                // secure as the secret is interpolated by Groovy, and so typical operating
                // system process listings will accidentally disclose it :
                sh ('''#!/usr/bin/env bash
                    # Stop previously launched container by image name (Works in docker 1.9+)
                     docker --host ${TG_HOST} ps -aqf "name=app" \
                        | xargs -r docker --host ${TG_HOST} stop

                    docker --host ${TG_HOST} run \
                        --env VIRTUAL_HOST=app.baridibaridi.com \
                        --name app \
                        --publish ${TG_PORT}:80 \
                        --detach \
                        --rm \
                        --privileged \
                    ${IMG_TAGGED}
                ''')
            }
        }
    }    
}


//////////////////////////////////////////////////////////////////////other expample
// Declarative pipeline
@Library('docker-deploy') _

pipeline {

  agent any

  environment {
    DOCKER_IMG     = "bb_iota_verification_api" // The image to push.
    SSH_HOST       = "0.0.0.0"   // The host to use
    SSH_USER       = "ec2-user"        // SSH user 
    CREDENTIALS_ID = "baridibaridi_app"  // ID of the credential
  }

  stages {

    stage("Build") {
      steps {
        // echo "Renaming the existing container bb_iota_verification_api to bb_iota_verification_api_old${currentBuild.startTimeInMillis}"
        // //renaming the existing container to avoid container repeationmn
        // sh "docker rename bb_iota_verification_api bb_iota_verification_api_old${currentBuild.startTimeInMillis}"

        sh "docker stop bb_iota_verification_api"
        sh "docker rm -f bb_iota_verification_api"
        // Build the image.
        sh "docker build -t ${env.DOCKER_IMG} ."
      }
    }

    stage("Deliver") {
      steps {
        // This will push the image `env.DOCKER_IMG` to the deployment host.
        dockerRemoteSave(credentialsId: env.CREDENTIALS_ID, img: env.DOCKER_IMG, host: env.SSH_HOST, user: SSH_USER)
      }
    }
  
    stage("Deploy") {
      environment {
        BIND_PORT      = 5000   // Your bind port - Exposed to other services public.
        CONTAINER_PORT = 3333   // Container port - Not accessible. to other service
        APP_NAME       = "bb_iota_verification_api"
        ENV_FILE_ID    = "bb_iota_verification_api_env"
      }

      steps {
        echo "Attempt to use envfile: ${env.ENV_FILE_ID}"

        dockerRemoteRun(
          host: "${env.SSH_HOST}",
          user: "${env.SSH_USER}",
          img: "${env.DOCKER_IMG}",
          bindPort: "${env.BIND_PORT}",
          containerPort: "${env.CONTAINER_PORT}",
          app: "${env.APP_NAME}",
          env: "${env.ENV_FILE_ID}"
        )
      }
    }
  }
}







