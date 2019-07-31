def label = "slave-${UUID.randomUUID().toString()}"
def helmLint(String chartDir) {
    println "校验 chart 模板"
    sh "helm lint ${chartDir}"
}

def helmInit() {
  println "初始化 helm client"
  sh "helm init --client-only --stable-repo-url https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts"
  sh "helm plugin install https://github.com/hypnoglow/helm-s3.git"
}

def helmRepo(Map args) {
  //println "添加 course repo"
  //sh "helm repo add --username ${args.username} --password ${args.password} course https://registry.qikqiak.com/chartrepo/course"
  println "添加 bixby-cndev-s3 repo"
  sh "helm repo add  bixby-cndev-s3 s3://cndevops-team-tmp/helm-repository-list"

  println "更新 repo"
  sh "helm repo update"

  println "获取 Chart 包"
  sh """
    helm fetch bixby-cndev-s3/polling
    tar -xzvf polling-0.1.0.tgz
    """
}

def helmDeploy(Map args) {
    helmInit()
    helmRepo(args)

    if (args.dry_run) {
        println "Debug 应用"
        sh "helm upgrade --dry-run --debug --install ${args.name} ${args.chartDir} --set persistence.persistentVolumeClaim.database.storageClass=awsebs --set api.image.repository=${args.image} --set api.image.tag=${args.tag} --set imagePullSecrets[0].name=myreg --namespace=${args.namespace}"
    } else {
        println "部署应用"
        sh "helm upgrade --install ${args.name} ${args.chartDir} --set persistence.persistentVolumeClaim.database.storageClass=awsebs --set api.image.repository=${args.image} --set api.image.tag=${args.tag} --namespace=${args.namespace}"
        echo "应用 ${args.name} 部署成功. 可以使用 helm status ${args.name} 查看应用状态"
    }
}


podTemplate(label: label,serviceAccount: 'jenkins2', containers: [
  containerTemplate(name: 'maven', image: 'maven:3.6-alpine', command: 'cat', ttyEnabled: true),
  containerTemplate(name: 'docker', image: 'docker', command: 'cat', ttyEnabled: true),
  containerTemplate(name: 'kubectl', image: 'cnych/kubectl', command: 'cat', ttyEnabled: true),
  containerTemplate(name: 'helm', image: 'zhaoxy8/helm-s3', command: 'cat', ttyEnabled: true)
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
          #kubectl apply -f manifests/k8s.yaml
        """
      }
    }
    stage('运行 Helm') {
      container('helm') {
            // todo，可以做分支判断
            echo "4. [INFO] 开始 Helm 部署"
            helmDeploy(
                dry_run     : false,
                name        : "polling",
                chartDir    : "polling",
                namespace   : "course",
                tag         : "${imageTag}",
                image       : "${image}"
				//username    : "${DOCKER_HUB_USER}", 私有chart仓库用户名
				//password    : "${DOCKER_HUB_PASSWORD}" 私有chart仓库密码
            )
            echo "[INFO] Helm 部署应用成功..."
      }
    }
  }
}
