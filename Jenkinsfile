#!/usr/bin/env groovy

library identifier: 'jenkins-shared-library@main', retriever: modernSCM(
    [$class: 'GitSCMSource',
     remote: 'https://github.com/nbglink/jenkins-shared-library.git',
     credentialsId: 'github-credentials'
    ]
)
// CI part
def user_choice = ""
pipeline {
    agent any
    stages {
    //Infrastructure provisioning
        stage('Provision an EKS Cluster') {
            steps {
                script {
                    dir("terraform-eks-infra") {
                        echo 'EKS Cluster provisioning...'
                        sh "terraform init"
                        sh "terraform apply --auto-approve"
                    }
                }
            }
        }
        stage('Update Kubeconfig') {
            steps {
                script {
                    dir("terraform-eks-infra") {
                        sh "aws eks --region \$(terraform output -raw region) update-kubeconfig --name \$(terraform output -raw cluster_name)"
                    }
                }
            }
        }
        stage("Install ArgoCD") {
            when {
                expression {
                    sh('kubectl get deployment argocd-server -n argocd -o jsonpath="{.metadata.labels.app}"') == "argocd-server"
                }
            }
            steps {
                sh "kubectl create namespace argocd"
                sh "kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml"
                sh "kubectl patch svc argocd-server -n argocd -p '{\"spec\": {\"type\": \"LoadBalancer\"}}'"
            }
        }
        stage('deploy') {
            steps {
                dir("argocd-app-config") {
                    sh "kubectl apply -f application.yaml"
                }
            }
        }
        stage('Destroy the EKS cluster? if Yes the clster will be destroyed') {
            steps {
                script {
                    user_choice = input message: 'Are you sure you want to execute this stage?', ok: 'Proceed', parameters: [choice(choices: 'Yes\nNo', description: '', name: 'user_choice')]
                }
            }
        }
        stage('Destroying EKS cluster?') {
            when {
                expression { user_choice == 'Yes' }
            }
            steps {
                echo 'EKS cluster destroyng...'
                dir("terraform-eks-infra") {
                        sh ""
                        sh "kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml"
                        sh "kubectl delete namespace argocd"
                        sh "terraform destroy --auto-approve"
                    }
            }
        }
        stage('Update GIT') {
          steps {
            script {
              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                withCredentials([usernamePassword(credentialsId: 'github-credentials', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                    def encodedPassword = URLEncoder.encode("$GIT_PASSWORD",'UTF-8')
                    sh "git add terraform-eks-infra/"
                    sh "git commit -m 'Triggered Build: ${env.BUILD_NUMBER}'"
                    sh "git push https://${GIT_USERNAME}:${encodedPassword}@github.com/${GIT_USERNAME}/DevOpsUpskill-FinalProject-Infra.git HEAD:main"
                }
              }
            }
          }
        }
    }
}
