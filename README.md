# Jenkins
Baisc Pipline steps


pipeline {
   
   agent any
   
    environment 
    {
        APP_NAME="my_app"
        YAML_PATH="other-java-services"

        CLUSTER1_DOCER_TAG_URL="docker-registry-default.com"
        CLUSTER1_DOCKER_REGISTRY="docker-registry.default.svc:5000"
		CLUSTER1_OC_URL="https://OC_URL"
		CLUSTER1_OC_USER=""
		CLUSTER1_OC_PASSWORD=""
		
		AWS_DOCER_TAG_URL="docker-registry-default.apps.com"
        AWS_DOCKER_REGISTRY="docker-registry.default.svc:5000"
		AWS_OC_URL="https://console.com"
		AWS_OC_USER=""
		AWS_OC_PASSWORD=""

    }
   
  stages
  { 

	   stage('ENV SET UP') 
	   {
	     steps 
	        {
			  
			     script 
			     {   
                		 if (env.PROJECT_NAME=='MY_PROJ1') {
                		      env.GIT_PROFILE="dev"
                		      env.GIT_LABEL="gold"
                		      env.ENVIRONMENT="CLUSTER1"
                		      env.TYPE="non-vault"
                		 } else if (env.PROJECT_NAME=='MY_PROJ2') {
                		      env.GIT_PROFILE="dev"
                		      env.GIT_LABEL="corp"
                		      env.ENVIRONMENT="CLUSTER1"
                		      env.TYPE="vault"
                		 }else if (env.PROJECT_NAME=='MY_PROJ3') {
                              env.GIT_PROFILE="aws"
                		      env.GIT_LABEL="aws"
                		      env.ENVIRONMENT="AWS"
                		      env.TYPE="vault"
                		 }
                		 
                		env.DOCER_TAG_URL="${ENVIRONMENT}_DOCER_TAG_URL"
		                env.DOCKER_REGISTRY="${ENVIRONMENT}_DOCKER_REGISTRY"
		                env.OC_URL="${ENVIRONMENT}_OC_URL"
		                env.OC_USER="${ENVIRONMENT}_OC_USER"
		                env.OC_PASSWORD="${ENVIRONMENT}_OC_PASSWORD"
                		 
            	}

		  }
	   }



	   stage('Download Confuguration') {
	     steps 
	      {
			   git branch: "${TYPE}" , credentialsId: '1fsdfasdfasdfsdfasdfasdff01d62a-8654-4647-a014-4f416c17cc3d', url: 'https://bitbucket.org/ankitpatwa/sdu-jenkin'
			 
		  }
	   }
	   
	  stage('Download Build') {
	     steps {
		     	sh  'curl  --fail $(sh getJfrogURL.sh $APP_NAME $APP_VERSION) >$APP_NAME-$APP_VERSION.jar'
		  }
	   }


	   stage('Build Docker') {
		steps {
			sh 'cp -r $YAML_PATH/* ./'        
			sh 'oc login --insecure-skip-tls-verify ${!OC_URL} -u ${!OC_USER}  -p ${!OC_PASSWORD} '
			sh 'oc project $PROJECT_NAME'
			sh 'docker login ${!DOCER_TAG_URL} -u dev -p $(oc whoami --show-token)'
			sh 'docker build -t  ${!DOCER_TAG_URL}/$PROJECT_NAME/$APP_NAME:${APP_VERSION}  --build-arg SERVICE_JAR=$APP_NAME-${APP_VERSION}.jar  .'
			sh 'docker push ${!DOCER_TAG_URL}/$PROJECT_NAME/$APP_NAME:${APP_VERSION}'
		}	
	   }

	   
		stage('Deploy') 
		{		 
			steps {			    
				sh 'oc delete service $APP_NAME-service -n  $PROJECT_NAME || true'
				sh 'oc delete  deploymentconfig $APP_NAME -n  $PROJECT_NAME || true'
				sh 'oc delete configmap $APP_NAME-config -n  $PROJECT_NAME || true'
				sh 'oc delete routes $APP_NAME -n  $PROJECT_NAME || true'
				sh 'oc new-app -f  configure.yml  --param PROJECT_NAME=$PROJECT_NAME  --param SERVICE_NAME=$APP_NAME --param SPRING_CLOUD_CONFIG_PROFILE=$GIT_PROFILE --param SPRING_CLOUD_CONFIG_LABEL=$GIT_LABEL -n  $PROJECT_NAME' 
				sh 'oc new-app -f deploy.yml --param PROJECT_NAME=$PROJECT_NAME  --param DOCKER_REGISTRY=${!DOCKER_REGISTRY}/$PROJECT_NAME --param SERVICE_VERSION=${APP_VERSION} --param IMAGE_NAME=$APP_NAME --param SERVICE_NAME=$APP_NAME  -n $PROJECT_NAME' 
			}
		}

     } 
 
     //Clear workspace after pipeline complete
       post { 
        always { 
            cleanWs()
        }
    }

}
