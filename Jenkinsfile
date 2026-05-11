pipeline {
  agent {
    kubernetes {
      label 'strategy-infaxia-build'
      yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: strategy-infaxia-build
spec:
  serviceAccountName: jenkins
  containers:
    - name: awscli
      image: public.ecr.aws/aws-cli/aws-cli:2.22.12
      command:
        - cat
      tty: true
      workingDir: /workspace
      volumeMounts:
        - name: workspace-volume
          mountPath: /workspace
  volumes:
    - name: workspace-volume
      emptyDir: {}
  tolerations:
    - key: scheduling/node
      operator: Equal
      value: jenkins-agent
      effect: NoSchedule
  nodeSelector:
    karpenter.sh/nodepool: node-jenkins-agent
"""
    }
  }

  options {
    skipStagesAfterUnstable()
    timestamps()
  }

  environment {
    AWS_CREDENTIALS_ID = 'aws-s3-credentials'
    AWS_DEFAULT_REGION = 'ap-southeast-1'
    S3_BUCKET = 'strategy-infaxia'
    S3_PREFIX = 'build-dev'
    CLOUDFRONT_DISTRIBUTION_ID = 'E1RN1W95XRI69Y'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Deploy to S3 and Invalidate CDN') {
      steps {
        container('awscli') {
          withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: env.AWS_CREDENTIALS_ID]]) {
            sh '''
              set -e
              SYNC_SOURCE="."
              if [ -d build ]; then
                echo "Found build/ directory. Deploying from build/."
                SYNC_SOURCE="build"
              else
                echo "No build/ directory. Deploying from current workspace (static files)."
              fi
              aws s3 sync "$SYNC_SOURCE" s3://$S3_BUCKET/$S3_PREFIX --delete --exclude "Jenkinsfile" --exclude "README.md"
              aws cloudfront create-invalidation --distribution-id $CLOUDFRONT_DISTRIBUTION_ID --paths "/*"
            '''
          }
        }
      }
    }
  }
}

