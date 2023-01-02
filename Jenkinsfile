pipeline {

    agent any

    options {
        ansiColor('xterm')
    }

    environment {
        IMG = "micro_service_api:${BUILD_ID}"
        IMG_TAGGED = "micro_service_api:${BUILD_ID}"
        TG_HOST = "ssh://ec2-user@13.245.2.166"
        TG_SSH  = "13.245.2.166"
        TG_USR  = "ec2-user"
        TG_PORT = 4000  //port exposed to external services(in proxy block configurations.)
    }
;
    stages {
        // Build the image
        stage("Build") {
            steps {
                // sh "docker stop micro_service_api"
                // sh "docker rm -f micro_service_api"
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
                // system process listings will accidentally disclose it
                //Note ~/dotenvs/.env located in /var/lib/jenkins/ directories for jenkins to access it.
                
                sh ('''#!/usr/bin/env bash
                    # Stop previously launched container by image name (Works in docker 1.9+)
                     docker --host ${TG_HOST} ps -aqf "name=micro_service_api" \
                        | xargs -r docker --host ${TG_HOST} stop

                    docker run --name  micro_service_api  --rm --detach \
                            --privileged  \
                            --env VIRTUAL_HOST=micro_service_api.baridibaridi.com \
                            --env-file ~/dotenvs/.env \
                            --publish ${TG_PORT}:4000 \
                    ${IMG_TAGGED}
                ''')
            }
        }
    }    
}
