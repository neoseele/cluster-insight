node {
  def project = 'nmiu-play'
  def appName = 'cluster-insight'
  def feSvcName = "${appName}"
  def imageTag = "gcr.io/${project}/${appName}:${env.BRANCH_NAME}.${env.BUILD_NUMBER}"

  checkout scm

  stage 'Build image'
  sh("docker build -t ${imageTag} collector")

  stage 'Push image to registry'
  sh("gcloud docker -- push ${imageTag}")

  stage "Deploy Application"
  switch (env.BRANCH_NAME) {
    // Roll out to canary or production environment
    case ["master"]:
        // Change deployed image in canary to the one we just built
        sh("sed -i.bak 's#kubernetes/cluster-insight:v2.0-alpha#${imageTag}#' ./install/cluster-insight-controller.yaml")
        sh("kubectl --namespace=prod apply -f install/cluster-insight-controller.yaml")
        sh("kubectl --namespace=prod apply -f install/cluster-insight-service.yaml")
        sh("echo http://`kubectl --namespace=prod get service/${feSvcName} --output=json | jq -r '.status.loadBalancer.ingress[0].ip'` > ${feSvcName}")
        break

    // Roll out a dev environment
    default:
        // Create namespace if it doesn't exist
        sh("kubectl get ns ${env.BRANCH_NAME} || kubectl create ns ${env.BRANCH_NAME}")
        // Don't use public load balancing for development branches
        sh("sed -i.bak 's#kubernetes/cluster-insight:v2.0-alpha#${imageTag}#' ./install/cluster-insight-controller.yaml")
        sh("kubectl --namespace=${env.BRANCH_NAME} apply -f install/cluster-insight-controller.yaml")
        sh("kubectl --namespace=${env.BRANCH_NAME} apply -f install/cluster-insight-service.yaml")
        sh("echo http://`kubectl --namespace=${env.BRANCH_NAME} get service/${feSvcName} --output=json | jq -r '.status.loadBalancer.ingress[0].ip'` > ${feSvcName}")
        echo 'To access your environment run `kubectl proxy`'
        echo "Then access your service via http://localhost:8001/api/v1/proxy/namespaces/${env.BRANCH_NAME}/services/${feSvcName}:80/"
  }
}
