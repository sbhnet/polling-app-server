def label = "slave-${UUID.randomUUID().toString()}"

podTemplate(label: label,serviceAccount: 'jenkins2', containers: [
  containerTemplate(name: 'maven', image: 'maven:3.6-alpine', command: 'cat', ttyEnabled: true),
  containerTemplate(name: 'docker', image: 'docker', command: 'cat', ttyEnabled: true),
  containerTemplate(name: 'kubectl', image: 'cnych/kubectl', command: 'cat', ttyEnabled: true),
  containerTemplate(name: 'helm', image: 'cnych/helm', command: 'cat', ttyEnabled: true)
], volumes: [
  hostPathVolume(mountPath: '/root/.m2', hostPath: '/var/run/m2'),
  hostPathVolume(mountPath: '/home/jenkins/.kube', hostPath: '/root/.kube'),
  hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock')
]) {
  node(label) {
    def myRepo = checkout scm
    def gitCommit = myRepo.GIT_COMMIT
    def gitBranch = myRepo.GIT_BRANCH
    def imageTag = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
    def dockerRegistryUrl = "docker.io"
    def imageEndpoint = "zhaoxy8/polling-app-server"
    def image = "${dockerRegistryUrl}/${imageEndpoint}"

    stage('单元测试') {
      echo "测试阶段"
    }
    stage('代码编译打包') {
      try {
        container('maven') {
          echo "2. 代码编译打包阶段"
          sh "mvn clean package -Dmaven.test.skip=true"
        }
      } catch (exc) {
        println "构建失败 - ${currentBuild.fullDisplayName}"
        throw(exc)
      }
    }
    stage('构建 Docker 镜像') {
      withCredentials([[$class: 'UsernamePasswordMultiBinding',
          credentialsId: 'dockerHub',
          usernameVariable: 'dockerHubUser',
          passwordVariable: 'dockerHubPassword']]) {
            container('docker') {
              echo "3. 构建 Docker 镜像阶段"
              sh """
                docker login -u ${dockerHubUser} -p ${dockerHubPassword}
                docker build -t ${image}:${imageTag} .
                docker push ${image}:${imageTag}
                """
            }
        }  
    }
    stage('运行 Kubectl') {
      container('kubectl') {
        echo "查看 K8S 集群 Pod 列表"
        sh "kubectl get pods"
        sh """
          sed -i "s#<IMAGE>#${image}#" manifests/k8s.yaml
          sed -i "s#<IMAGE_TAG>#${imageTag}#" manifests/k8s.yaml
          kubectl apply -f k8s.yaml
        """
      }
    }
    stage('运行 Helm') {
      container('helm') {
        echo "查看 Helm Release 列表 "
        sh "helm list"
      }
    }
  }
}
