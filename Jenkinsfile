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
                            slackSend color: 'good', message: "Provision an EKS Cluster - SUCCESS: EKS cluster has been successfully provisioned!"
                        } catch (Exception e) {
                            error("EKS Cluster provisioning failed with error: ${e.getMessage()}")
                            slackSend color: 'danger', message: "Provision an EKS Cluster - FAILURE: There was an issue while provisioning EKS cluster with error: ${e.getMessage()}"
                        }
                    }
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
                            slackSend color: 'good', message: "Update GIT to update terraform status files in the repo - SUCCESS: Git repository has been successfully updated!"
                        }
                    }
                }
            }
        }
        stage("Install ArgoCD") {
            steps {
                script {
                    try {
                        sh "kubectl get namespace argocd"
                        slackSend color: 'good', message: "Install ArgoCD - SUCCESS: ArgoCD namespace already exists!"
                    } catch (Exception e) {
                        sh "kubectl create namespace argocd"
                        sh "kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml"
                        sh "kubectl patch svc argocd-server -n argocd -p '{\"spec\": {\"type\": \"LoadBalancer\"}}'"
                        slackSend color: 'good', message: "Install ArgoCD - SUCCESS: ArgoCD has been installed!"
                    }
                }
            }
        }
        stage('Destroying EKS cluster') {
            when {
                expression { user_choice == 'Yes' }
            }
            steps {
                echo 'EKS cluster destroying...'
                dir("terraform-eks-infra") {
                    try {
                        sh "kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml"
                        sh "kubectl delete namespace argocd"
                        sh "terraform destroy --auto-approve"
                        slackSend color: 'good', message: "Destroying EKS cluster - SUCCESS: EKS cluster has been successfully destroyed!"
                    } catch (Exception e) {
                        error("EKS Cluster destruction failed with error: ${e.getMessage()}")
                        slackSend color: 'danger', message: "Destroying EKS cluster - FAILURE: There was an issue while destroying EKS cluster with error: ${e.getMessage()}"
                    }
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
                            slackSend color: 'good', message: "Update GIT to clean the terraform status files in the repo after destroy - SUCCESS: Git repository has been successfully updated!"
                        }
                    }
                }
            }
        }
    }
}
