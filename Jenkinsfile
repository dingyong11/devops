pipeline {

  agent {
      # 定义使用 Kubernetes 作为 agent
      kubernetes {
      # 选择的云为之前配置的名字
        cloud 'kubernetes-study'
        slaveConnectTimeout 1200
      # 将 workspace 改成 hostPath，因为该 Slave 会固定节点创建，如果有存储可用，可以改成 PVC 的模式
        workspaceVolume hostPathWorkspaceVolume(hostPath: "/opt/workspace", readOnly: false)
        yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  # jnlp 容器，和 Jenkins 主节点通信
  - args: [\'$(JENKINS_SECRET)\', \'$(JENKINS_NAME)\']
    image: 'registry.cn-beijing.aliyuncs.com/citools/jnlp:alpine'
    name: jnlp
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - mountPath: "/etc/localtime"
      name: "localtime"
      readOnly: false 
  # build 容器，包含执行构建的命令，比如 Java 的需要 mvn 构建，就可以用一个 maven 的镜像 
  - command:
    - "cat"
    env:
    - name: "LANGUAGE"
      value: "en_US:en"
    - name: "LC_ALL"
      value: "en_US.UTF-8"
    - name: "LANG"
      value: "en_US.UTF-8"
    image: "registry.cn-beijing.aliyuncs.com/citools/maven:3.5.3"
    imagePullPolicy: "IfNotPresent"
    name: "build"
    tty: true
    volumeMounts:
    - mountPath: "/etc/localtime"
      name: "localtime"
      # Pod 单独创建了一个缓存的 volume，将其挂载到了 maven 插件的缓存目录，默认是/root/.m2
    - mountPath: "/root/.m2/"
      name: "cachedir"
      readOnly: false
  # 发版容器，因为最终是发版至 Kubernetes 的，所以需要有一个 kubectl 命令
  - command:
    - "cat"
    env:
    - name: "LANGUAGE"
      value: "en_US:en"
    - name: "LC_ALL"
      value: "en_US.UTF-8"
    - name: "LANG"
      value: "en_US.UTF-8"
  # 镜像的版本可以替换为其它的版本，也可以不进行替换，因为只执行 set 命令，所以版本是兼容的
    image: "registry.cn-beijing.aliyuncs.com/citools/kubectl:self-1.17"
    imagePullPolicy: "IfNotPresent"
    name: "kubectl"
    tty: true
    volumeMounts:
    - mountPath: "/etc/localtime"
      name: "localtime"
      readOnly: false
  # 用于生成镜像的容器，需要包含 docker 命令
  - command:
    - "cat"
    env:
    - name: "LANGUAGE"
      value: "en_US:en"
    - name: "LC_ALL"
      value: "en_US.UTF-8"
    - name: "LANG"
      value: "en_US.UTF-8"
      image: "registry.cn-beijing.aliyuncs.com/citools/docker:19.03.9-git"
      imagePullPolicy: "IfNotPresent"
      name: "docker"
      tty: true
      volumeMounts:
      - mountPath: "/etc/localtime"
        name: "localtime"
        readOnly: false
  # 由于容器没有启动 docker 服务，所以将宿主机的 docker 经常挂载至容器即可
      - mountPath: "/var/run/docker.sock"
        name: "dockersock"
        readOnly: false
  restartPolicy: "Never"
  # 固定节点部署
  nodeSelector:
    build: "true"
  securityContext: {}
  volumes:
  # Docker socket volume
  - hostPath:
      path: "/var/run/docker.sock"
    name: "dockersock"
  - hostPath:
      path: "/usr/share/zoneinfo/Asia/Shanghai"
    name: "localtime"
  # 缓存目录
  - name: "cachedir"
    hostPath:
      path: "/opt/m2"
'''
    }
 }

stages {

  stage('Pulling Code') {
    parallel {
       stage('Pulling Code by Jenkins') {
         when {
           expression {
           # 假如 env.gitlabBranch 为空，则该流水线为手动触发，那么就会执行该stage，如果不为空则会执行同级的另外一个 stage
             env.gitlabBranch == null
           }
         }
         steps {
           # 这里使用的是 git 插件拉取代码，BRANCH 变量取自于前面介绍的 parameters配置
           # git@xxxxxx:root/spring-boot-project.git 代码地址
           # credentialsId: 'gitlab-key'，之前创建的拉取代码的 key
           git(changelog: true, poll: true, url: 'git@xxxxxx:root/springboot-project.git', branch: "${BRANCH}", credentialsId: 'gitlab-key')
           script {  
             # 定义一些变量用于生成镜像的 Tag
             # 获取最近一次提交的 Commit ID
             COMMIT_ID = sh(returnStdout: true, script: "git log -n 1 --pretty=format:'%h'").trim()
             # 赋值给 TAG 变量，后面的 docker build 可以取到该 TAG 的值
             TAG = BUILD_TAG + '-' + COMMIT_ID
             println "Current branch is ${BRANCH}, Commit ID is ${COMMIT_ID}, Image TAG is ${TAG}"
           }
         }
       }
       stage('Pulling Code by trigger') {
         when {
           expression {
           # 如果 env.gitlabBranch 不为空，说明该流水线是通过 webhook 触发，执行该 stage，上述的 stage 不再执行。此时 BRANCH 变量为空
             env.gitlabBranch != null
           }
         }
         steps {
           # 以下配置和上述一致，只是此时 branch: env.gitlabBranch 取的值为env.gitlabBranch
           git(url: 'git@xxxxxxxxxxx:root/spring-boot-project.git', branch: env.gitlabBranch, changelog: true, poll: true, credentialsId: 'gitlab-key')
           script {
             COMMIT_ID = sh(returnStdout: true, script: "git log -n 1 --pretty=format:'%h'").trim()
             TAG = BUILD_TAG + '-' + COMMIT_ID
             println "Current branch is ${env.gitlabBranch}, Commit ID is ${COMMIT_ID}, Image TAG is ${TAG}"
           }
         }
       }
     }
 }



  
}
接下来是拉代码的 stage，这个 stage 是一个并行的 stage，因为考虑了该流水线是手动触
发还是触发：

代码拉下来后，就可以执行构建命令，由于本次实验是 Java 示例，所以需要使用 mvn 命令
进行构建：
stage('Building') {
 steps {
 # 使用 Pod 模板里面的 build 容器进行构建
 container(name: 'build') {
 sh """ 
# 编译命令， 需要根据自己项目的实际情况进行修改，可能会不一致
 mvn clean install -DskipTests
# 构建完成后，一般情况下会在 target 目录下生成 jar 包
 ls target/*
 """
 }
 }
 }
生成编译产物后，需要根据该产物生成对应的镜像，此时可以使用 Pod 模板的 docker 容器：
stage('Docker build for creating image') {
 # 首先取出来 HARBOR 的账号密码
environment {
 HARBOR_USER = credentials('HARBOR_ACCOUNT')
 }
 steps {
 # 指定使用 docker 容器
 container(name: 'docker') {
 sh """
 # 执行 build 命令，Dockerfile 会在下一小节创建，也是放在代码仓库，和
Jenkinsfile 同级
 docker build -t 
${HARBOR_ADDRESS}/${REGISTRY_DIR}/${IMAGE_NAME}:${TAG} .
# 登录 Harbor，HARBOR_USER_USR 和 HARBOR_USER_PSW 由上述 environment 生
成
 docker login -u ${HARBOR_USER_USR} -p ${HARBOR_USER_PSW} 
${HARBOR_ADDRESS}
# 将镜像推送至镜像仓库
 docker push 
${HARBOR_ADDRESS}/${REGISTRY_DIR}/${IMAGE_NAME}:${TAG}
 """
 }
 }
 }
最后一步就是将该镜像发版至 Kubernetes 集群中，此时使用的是包含 kubectl 命令的容器：
stage('Deploying to K8s') {
 # 获取连接 Kubernetes 集群证书 
environment {
 MY_KUBECONFIG = credentials('study-k8s-kubeconfig')
 }
 steps {
 # 指定使用 kubectl 容器
 container(name: 'kubectl'){
 sh """
 # 直接 set 更改 Deployment 的镜像即可
 /usr/local/bin/kubectl --kubeconfig $MY_KUBECONFIG set image 
deploy -l app=${IMAGE_NAME} 
${IMAGE_NAME}=${HARBOR_ADDRESS}/${REGISTRY_DIR}/${IMAGE_NAME}:${TAG} -n 
$NAMESPACE
 """
 }
 }
 }
 }
