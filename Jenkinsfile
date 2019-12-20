pipeline {
    agent {
        node { label 'aws && build && linux && ubuntu'}
    }
	parameters {
        string(name: 'BUILD_VERSION', defaultValue: '', description: 'The build version to deploy (optional)')
	    choice(name: 'DEPLOY_TO', choices: 'CI\nTEST', description: 'The deployment stage to trigger')
    }
    environment {
        PROJECT_NAME = "l"
        TYPE = "service"
    }
	stages {
	    stage('Build Version') {
            steps {
                script {
                    BUILD_VERSION_GENERATED = VersionNumber(
                        versionNumberString: 'v${BUILD_YEAR, XX}.${BUILD_MONTH, XX}${BUILD_DAY, XX}.${BUILDS_TODAY}',
                        projectStartDate:    '1970-01-01',
                        skipFailedBuilds:    true
                    )
                    currentBuild.displayName = BUILD_VERSION_GENERATED
                    env.BUILD_VERSION = BUILD_VERSION_GENERATED
                          }
                      }
                  }
	stage('Creating Reverse proxy') {
            steps {
                
               sh 'docker network inspect reverse-proxy &>/dev/null || docker network create reverse-proxy'
                    }
               }
	    stage('Push Images to AWS ECR') {
	        steps {
		        script {
                    docker.build("l_fluentd", "--no-cache ./fluentd/")
                    docker.build("l_kibana", "--no-cache ./kibana/")
                    docker.build("l_elasticsearch_node", "--no-cache ./elasticsearch/")
                    docker.build("l_elasticsearch", "--no-cache ./elasticsearch/")
                    docker.withRegistry('https://684150170045.dkr.ecr.us-east-1.amazonaws.com', 'ecr:us-east-1:aws-jenkins-build') {
                        docker.image("l_fluentd").push("current")
                        docker.image("l_kibana").push("current")
                        docker.image("l_elasticsearch_node").push("current")
                        docker.image("l_elasticsearch").push("current")
                                    }
                              }
                         }
                    }
		stage('Deploy - CI') {
            when {
                expression {
                    return env.DEPLOY_TO == 'CI'
                }
            }
	        agent {
                 label 'aws-ci-efk-server'
            }
            steps {
                configFileProvider([
		            configFile(fileId: 'esnode-key.pem', targetLocation: './elasticsearch/config/esnode-key.pem'),
		            configFile(fileId: 'esnode.pem', targetLocation: './elasticsearch/config/esnode.pem'),
		            configFile(fileId: 'kirk-key.pem', targetLocation: './elasticsearch/config/kirk-key.pem'),
		            configFile(fileId: 'kirk.pem', targetLocation: './elasticsearch/config/kirk.pem'),
		            configFile(fileId: 'root-ca.pem', targetLocation: './elasticsearch/config/root-ca.pem')
                ]) {
                    withAWS(credentials:'aws-jenkins-build') {
                        sh '''
                            export DOCKER_LOGIN="`aws ecr get-login --no-include-email --region us-east-1`"
                            $DOCKER_LOGIN
                        '''
                        ecrLogin()
				        script {
				            sh "sed -i 's/KIBANAHOST_VALUE/l.ci.aws.labshare.org/g' docker-compose.yml"
				            sh "sed -i 's/FLUENTDHOST_VALUE/fl.ci.aws.labshare.org/g' docker-compose.yml"
				            sh 'docker-compose -p $PROJECT_NAME down -v --rmi all | xargs echo'
				            sh 'chmod 644 ./elasticsearch/config/*.pem'
					    sh 'chmod 755 ./apm-server/config/apm-server.yml'
                            sh 'docker-compose -p $PROJECT_NAME up -d'
			            }
                               }
                          }
                     }
                }
	    stage('Deploy - TEST') {
            when {
                expression {
                    return env.DEPLOY_TO == 'TEST'
                }
            }
    	    agent {
                label 'aws-test-efk-server'
            }
            steps {
                configFileProvider([
    		        configFile(fileId: 'esnode-key.pem', targetLocation: './elasticsearch/config/esnode-key.pem'),
    		        configFile(fileId: 'esnode.pem', targetLocation: './elasticsearch/config/esnode.pem'),
    		        configFile(fileId: 'kirk-key.pem', targetLocation: './elasticsearch/config/kirk-key.pem'),
    		        configFile(fileId: 'kirk.pem', targetLocation: './elasticsearch/config/kirk.pem'),
    		        configFile(fileId: 'root-ca.pem', targetLocation: './elasticsearch/config/root-ca.pem')
                ]) {
                    withAWS(credentials:'aws-jenkins-build') {
                        sh '''
                            export DOCKER_LOGIN="`aws ecr get-login --no-include-email --region us-east-1`"
                            $DOCKER_LOGIN
                        '''
                        ecrLogin()
    				    script {
    				        sh "sed -i 's/KIBANAHOST_VALUE/l.test.aws.labshare.org/g' docker-compose.yml"
    				        sh "sed -i 's/FLUENTDHOST_VALUE/fl.test.aws.labshare.org/g' docker-compose.yml"
    				        sh 'docker-compose -p $PROJECT_NAME down -v --rmi all | xargs echo'
    				        sh 'chmod 644 ./elasticsearch/config/*.pem'
					sh 'chmod 755 ./apm-server/config/apm-server.yml'
                            sh 'docker-compose -p $PROJECT_NAME up -d'
                        }
                    }
                }
            }
        }
    }
}
