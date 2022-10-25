pipeline {
  environment {
    PROJECT = "phonic-realm-360311"
    APP_NAME = "gceme"
    FE_SVC_NAME = "${APP_NAME}-frontend"
    CLUSTER = "jenkins-cd"
    CLUSTER_ZONE = "asia-northeast3-a"
    IMAGE_TAG = "gcr.io/${PROJECT}/${APP_NAME}:${env.BRANCH_NAME}.${env.BUILD_NUMBER}"
    JENKINS_CRED = "${PROJECT}"
  }
  
  agent {
    kubernetes {
      yaml '''
      apiVersion: v1
      kind: Pod
      metadata:
        labels:
          app: blue-green-deploy
        name: blue-green-deploy
      spec:
        containers:
        - name: kustomize
          image: sysnet4admin/kustomize:3.6.1
          tty: true
          volumeMounts:
          - mountPath: /usr/bin/kubectl
            name: kubectl
          command:
          - cat
        serviceAccount: cd-jenkins
        volumes:
        - name: kubectl
          hostPath:
            path: /usr/bin/kubectl
      '''
    }
  }
  stages {
    stage('git scm update'){
      steps {
        git url: 'https://github.com/IaC-Source/blue-green.git', branch: 'main'
      }
    }
    stage('define tag'){
      steps {
        script {
          if(env.BUILD_NUMBER.toInteger() % 2 == 1){
            env.tag = "blue"
          } else {
            env.tag = "green"
          }
        }
      }
    }
    stage('deploy configmap and deployment'){
      steps {
        container('kustomize'){
          dir('deployment'){
            sh '''
            kubectl apply -f configmap.yaml
            kustomize create --resources ./deployment.yaml
            echo "deploy new deployment"
            kustomize edit add label deploy:$tag -f
            kustomize edit set namesuffix -- -$tag
            kustomize edit set image sysnet4admin/dashboard:$tag
            kustomize build . | kubectl apply -f -
            echo "retrieve new deployment"
            kubectl get deployments -o wide
            '''
          }
        }
      }    
    }
    stage('switching LB'){
      steps {
        container('kustomize'){
          dir('service'){
            sh '''
            kustomize create --resources ./lb.yaml
            while true;
            do
              export replicas=$(kubectl get deployments \
              --selector=app=dashboard,deploy=$tag \
              -o jsonpath --template="{.items[0].status.replicas}")
              export ready=$(kubectl get deployments \
              --selector=app=dashboard,deploy=$tag \
              -o jsonpath --template="{.items[0].status.readyReplicas}")
              echo "total replicas: $replicas, ready replicas: $ready"
              if [ "$ready" -eq "$replicas" ]; then
                echo "tag change and build deployment file by kustomize" 
                kustomize edit add label deploy:$tag -f
                kustomize build . | kubectl apply -f -
                echo "delete $tag deployment"
                kubectl delete deployment --selector=app=dashboard,deploy!=$tag
                kubectl get deployments -o wide
                break
              else
                sleep 1
              fi
            done
            '''
          }
        }
      }
    }
  }
}
