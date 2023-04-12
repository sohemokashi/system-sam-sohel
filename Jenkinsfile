version="1.0.0"
repository="docker-registry.mtk8s.io/backstage/backstage"
tag="latest"
image="${repository}:${version}.${env.BUILD_NUMBER}"
namespace="acnbackstage"

podTemplate(label: 'demo-customer-pod', cloud: 'kubernetes', serviceAccount: 'jenkins', imagePullSecrets: ['jenmtk8sdkrreg'], 
  containers: [
    containerTemplate(name: 'node16py311', image: 'nikolaik/python-nodejs:python3.11-nodejs16-bullseye', ttyEnabled: true, command: 'cat'),
    containerTemplate(name: 'buildkit', image: 'docker-registry.mtk8s.io/moby/buildkit:latest', ttyEnabled: true, privileged: true, alwaysPullImage: true),
    containerTemplate(name: 'kubectl', image: 'roffe/kubectl', ttyEnabled: true, command: 'cat'),
  ],
  volumes: [
    // secretVolume(mountPath: '/etc/.ssh', secretName: 'ssh-home'),
    secretVolume(secretName: 'jenkins-docker-creds', mountPath: '/root/.docker'),
    configMapVolume(configMapName: 'kube-root-ca.crt', mountPath: '/var/tmp'),
    configMapVolume(configMapName: 'kube-intermediate-ca.crt', mountPath: '/var/tmp')
    //configMapVolume(configMapName: 'docker-config', mountPath: '/root/.docker').
  ]) {

    node('demo-customer-pod') {
        stage('Prepare') {
            git changelog: false, credentialsId: 'github-app-jenkins', poll: false, branch: 'main', url: 'https://github.com/kbmdev/backstage-acn-repo.git'
            echo "SCM Checkout by using GitHub App"
        }
        
        stage('Node App Build and Package') {
            container('node16py311') {
                sh """
                  cd backstage-app
                  echo "List all the files in the PodTemplate Container from /var/tmp"
                  ls -latrh -R /var/tmp/
                  echo "/usr/local/share: List all the files in the PodTemplate Container"
                  ls -latrh -R /usr/local/share
                  yarn install
                  #yarn tsc
                  yarn build:backend
                  ls -latrh
                """
                milestone(1)
            }
        }

        stage('Build Docker Image') {
            container('buildkit') {
                sh """
                  ls -latrh
                  cd backstage-app/packages/backend
                  cp /usr/local/share/ca-certificates/kx-intermediate-ca.crt .
                  cp /usr/local/share/ca-certificates/kx-root-ca.crt .
                  
                  cp /usr/local/share/ca-certificates/kx-intermediate-ca.crt ../../
                  cp /usr/local/share/ca-certificates/kx-root-ca.crt ../../

                  ls -latrh
                  echo "Prior Path"
                  ls -latrh ../../
                  cat Dockerfile
                  ls -latrh -R /root/.docker
                  hostname
                  cat /etc/hosts
                  cat /etc/resolv.conf
                  whoami
                  pwd
                  buildctl --debug build --frontend dockerfile.v0 --progress=plain --local context=../.. --local dockerfile=. --output type=image,name=${image},push=true
                  buildctl --debug build --frontend dockerfile.v0 --progress=plain --local context=../.. --local dockerfile=. --output type=image,name=${repository}:${tag},push=true
                """
                milestone(2)
            }
        }

        stage('Deploy Latest') {
            container('kubectl') {
                echo "Skip Deploy Stage - Need to incorporate Helm or Kubectl"
                sh "kubectl patch -n ${namespace} deployment backstage-1679696953 -p '{\"spec\": { \"template\" : {\"spec\" : {\"containers\" : [{ \"name\" : \"backstage-backend\", \"image\" : \"${image}\"}]}}}}'"
                milestone(3)
            }
        }
    }
}

properties([[
    $class: 'BuildDiscarderProperty',
    strategy: [
        $class: 'LogRotator',
        artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '', numToKeepStr: '10']
    ]
]);