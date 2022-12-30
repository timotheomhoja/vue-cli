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
        TG_HOST = "ssh://ec2-user@13.245.2.166"
        TG_SSH  = "13.245.2.166"
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







