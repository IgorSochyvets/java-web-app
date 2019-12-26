#!/usr/bin/env groovy

env.DOCKERHUB_IMAGE = 'fizz-buzz'
env.DOCKERHUB_USER = 'kongurua'
//env.DEV_RELEASE_TAG = 'dev'
//env.QA_RELEASE_TAG = 'qa'
//env.PROD_RELEASE_TAG = 'prod'
//env.APPNAME = 'javawebapp'

def label = "jenkins-agent"

podTemplate(label: label, yaml: """
apiVersion: v1
kind: Pod
metadata:
  name: jenkins-slave
  namespace: jenkins
  labels:
    component: ci
    jenkins: jenkins-agent
spec:
  # Use service account that can deploy to all namespaces
  serviceAccountName: jenkins
  volumes:
  - name: dind-storage
    emptyDir: {}
  containers:
  - name: git
    image: alpine/git
    command:
    - cat
    tty: true
  - name: maven
    image: maven:latest
    command:
    - cat
    tty: true
  - name: kubectl
    image: lachlanevenson/k8s-kubectl:v1.8.8
    command:
    - cat
    tty: true
  - name: docker
    image: docker:19-git
    command:
    - cat
    tty: true
    env:
    - name: DOCKER_HOST
      value: tcp://docker-dind:2375
    volumeMounts:
      - name: dind-storage
        mountPath: /var/lib/docker
  - name: helm
    image: lachlanevenson/k8s-helm:v2.16.1
    command:
    - cat
    tty: true
"""
  ){

    node(label) {
      
      def tagDockerImage
      def nameStage

      stage('Checkout SCM') {
        checkout scm
      }

      stage('Unit Tests') {
        container('maven') {
          sh "mvn test" ;
          }
        }

      stage('Building Application') {
        container('maven') {
          sh "mvn install"
          }
        }

// Docker Image Building
        // Environment variables DOCKERHUB_USER, DOCKERHUB_IMAGE
        // var info from Jenkins plugins:
        // BRANCH_NAME = master  - master branch
        // BRANCH_NAME = PR-1    - pull request
        // BRANCH_NAME = develop - other branch
        // BRANCH_NAME = v0.0.1  - git tag
        //
    stage('Docker build') {
            container('docker') {
             withCredentials([usernamePassword(credentialsId: 'docker_hub_login', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASSWORD')]) {
               sh  'echo "Create Docker image: ${DOCKERHUB_IMAGE}:${BRANCH_NAME}"'
               sh  'docker login --username ${DOCKER_USER} --password ${DOCKER_PASSWORD}'
               sh  'docker build -t ${DOCKERHUB_USER}/${DOCKERHUB_IMAGE}:${BRANCH_NAME} .'
              }
            }
        }

// do not push docker image for PR
        if ( isPullRequest() ) {
            // exitAsSuccess()
            return 0
        }

// push docker image for all other cases (except PR)
        stage ('Docker push') {
            container('docker') {
              withCredentials([usernamePassword(credentialsId: 'docker_hub_login', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASSWORD')]) {
                    sh 'docker image ls'
                    sh  'docker push ${DOCKERHUB_USER}/${DOCKERHUB_IMAGE}:${BRANCH_NAME}'
              }
            }
        }

// do not deploy when 'push to branch' (and PR)
        if ( isPushtoFeatureBranch() ) {
                // exitAsSuccess()
                return 0
        }

// deploy


//Deploy to Master (Dev and Prod)
// DEV release
/*
        if ( isMaster() ) {
           stage('Deploy DEV release') {
                echo "Every commit to master branch is a dev release"
                echo "Deploy Dev release after commit to master"
                deployHelm("javawebapp-dev","dev",env.BRANCH_NAME)
           }

// PROD release
            if ( isChangeSet()  ) {

              stage('Deploy PROD release') {
                  echo "Production release controlled by a change to production-release.txt file in application repository root,"
                  echo "containing a git tag that should be released to production environment"

                  tagDockerImage = "${sh(script:'cat production-release.txt',returnStdout: true)}"
                  //? need check is tag exist

                  deployHelm("javawebapp-prod","prod",tagDockerImage)
                              // image tag from file production-release.txt , namespace , name chart release

              } //stage
        }


      } // end of Master block
*/

// PROD release
              if ( isMaster() ) {
                  if ( isChangeSet()  ) {
                    stage('Deploy PROD release') {
                        echo "Production release controlled by a change to production-release.txt file in application repository root,"
                        echo "containing a git tag that should be released to production environment"
                        tagDockerImage = "${sh(script:'cat production-release.txt',returnStdout: true)}"
                        //? need check is tag exist
                        deployHelm("javawebapp-prod","prod",tagDockerImage)
                                    // image tag from file production-release.txt , namespace , name chart release
                    } //stage
              }
            } // end of Master block






//Deploy QA with tag
            if ( isBuildingTag() ){
              stage('Deploy QA release') {
                  echo "Every git tag on a master branch is a QA release"

                  deployHelm( "javawebapp-qa",                      // name chart release
                              "qa",                           // namespace
                              env.BRANCH_NAME )               // image tag = 0.0.1

                          }

                      // integrationTest
                      // stage('approve'){ input "OK to go?" }
                      }


    } // node
  } //podTemplate



  // is it push to Master branch?
  def isMaster() {
      return (env.BRANCH_NAME == "master" )
  }

  def isPullRequest() {
      return (env.BRANCH_NAME ==~  /^PR-\d+$/)
  }

  def isBuildingTag() {
      // add check that  is branch master?
      return ( env.BRANCH_NAME ==~ /^\d.\d.\d$/ )
  }

  def isPushtoFeatureBranch() {
      return ( ! isMaster() && ! isBuildingTag() && ! isPullRequest() )
  }

  def isChangeSet() {

      def changeLogSets = currentBuild.changeSets
             for (int i = 0; i < changeLogSets.size(); i++) {
             def entries = changeLogSets[i].items
             for (int j = 0; j < entries.length; j++) {
                 def files = new ArrayList(entries[j].affectedFiles)
                 for (int k = 0; k < files.size(); k++) {
                     def file = files[k]
                     if (file.path.equals("production-release.txt")) {
                         return true
                     }
                 }
              }
      }

      return false
  }

/* check k8s commectivity
  def deploy( tagName, appName ) {

          echo "Release image: ${DOCKERHUB_IMAGE}:$tagName"
          echo "Deploy app name: $appName"

          withKubeConfig([credentialsId: 'kubeconfig']) {
          sh"""
              kubectl get ns
          """
          }

  }
  */
//
// name = javawebapp
// ns = dev/qa/prod
// tag = image's tag

// !! need to change deployment version or label in order to re-deploy pod
  def deployHelm(name, ns, tag) {
     container('helm') {
        withKubeConfig([credentialsId: 'kubeconfig']) {
        sh """
            echo "Deployments is starting..."
            helm delete $name

            helm upgrade --install $name --debug ./javawebapp-chart \
            --force \
            --wait \
            --namespace $ns \
            --set image.tag=$tag \
            --set image.repository=$DOCKERHUB_USER/$DOCKERHUB_IMAGE \
            --set ingress.hostName=${name}.ddns.net \
            --set-string ingress.hosts[0].host=${name}.ddns.net \
            --set-string ingress.tls[0].hosts[0]=${name}.ddns.net \
            --set-string ingress.tls[0].secretName=acme-$name-tls

            helm ls
        """

        }
    }

}

/*
--set-string ingress.tls[0].hosts[0]=$name.$host \
--set-string ingress.tls[0].secretName=acme-$name-tls
*/
