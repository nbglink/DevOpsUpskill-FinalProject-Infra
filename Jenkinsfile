#!/usr/bin/env groovy

library identifier: 'jenkins-shared-library@main', retriever: modernSCM(
    [$class: 'GitSCMSource',
     remote: 'https://github.com/nbglink/jenkins-shared-library.git',
     credentialsId: 'github-credentials'
    ]
)
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
                        try {
                            sh "terraform apply --auto-approve"
                        } catch (Exception e) {
                            error("EKS Cluster provisioning failed with error: ${e.getMessage()}")
                        }
                    }
                }
            }
            post {
                success {
                    slackSend color: 'good', message: "Provision an EKS Cluster - SUCCESS: EKS cluster has been successfully provisioned!"
                }
                failure {
                    slackSend color: 'danger', message: "Provision an EKS Cluster - FAILURE: There was an issue while provisioning EKS cluster!"
                }
            }
        }
        stage('Update GIT to update terraform status files in the repo') {
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
            post {
                success {
                    slackSend color: 'good', message: "Update GIT to update terraform status files in the repo - SUCCESS: Git repository has been successfully updated!"
                }
                failure {
                    slackSend color: 'danger', message: "Update GIT to update terraform status files in the repo - FAILURE: There was an issue while updating Git repository!"
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
            post {
                success {
                    slackSend color: 'good', message: "Update Kubeconfig - SUCCESS: Kubeconfig has been successfully updated!"
                }
                failure {
                    slackSend color: 'danger', message: "Update Kubeconfig - FAILURE: There was an issue while updating Kubeconfig!"
                }
            }
        }
        stage("Install ArgoCD") {
            steps {
                script {
                    try {
                        sh "kubectl get namespace argocd"
                    } catch (Exception e) {
                        sh "kubectl create namespace argocd"
                        sh "kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml"
                        sh "kubectl patch svc argocd-server -n argocd -p '{\"spec\": {\"type\": \"LoadBalancer\"}}'"
                    }
                }
            }
            post {
                success {
                    slackSend color: 'good', message: "Install ArgoCD - SUCCESS: ArgoCD has been successfully installed!"
                }
                failure {
                    slackSend color: 'danger', message: "Install ArgoCD - FAILURE: There was an issue while installing ArgoCD!"
                }
            }
        }
        stage('Destroy the EKS cluster? if Yes the clster will be destroyed') {
            steps {
                script {
                    user_choice = input message: 'Are you sure you want to execute this stage?', ok: 'Proceed', parameters: [choice(choices: 'No\nYes', description: 'Be careful with this step!!!', name: 'user_choice')]
                }
            }
        }
        stage('Destroying EKS cluster') {
            when {
                expression { user_choice == 'Yes' }
            }
            steps {
                echo 'EKS cluster destroyng...'
                dir("terraform-eks-infra") {
                        sh "kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml"
                        sh "kubectl delete namespace argocd"
                        sh "terraform destroy --auto-approve"
                    }
            }
        }
        stage('Update GIT to clean the terraform status files in the repo after destroy') {
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
