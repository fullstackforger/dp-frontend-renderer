#!groovy

node {
    def gopath = pwd()

    ws("${gopath}/src/github.com/ONSdigital/dp-frontend-renderer") {
        stage('Checkout') {
            checkout scm
            sh 'git clean -dfx'
            sh 'git rev-parse --short HEAD > git-commit'
            sh 'set +e && (git describe --exact-match HEAD || true) > git-tag'
        }

        def revision = revisionFrom(readFile('git-tag').trim(), readFile('git-commit').trim())

        stage('Build') {
            sh "GOPATH=${gopath} go build -o build/dp-frontend-renderer"
        }

        stage('Test') {
            sh "GOPATH=${gopath} go test ./..."
        }

        stage('Image') {
            docker.withRegistry("https://${env.ECR_REPOSITORY_URI}", { ->
                sh '$(aws ecr get-login)'
                docker.build('frontend-renderer', '--no-cache --pull --rm .').push(revision)
            })
        }

        stage('Bundle') {
            sh sprintf('sed -i -e %s -e %s -e %s -e %s -e %s appspec.yml scripts/codedeploy/*', [
                "s/\\\${CODEDEPLOY_USER}/${env.CODEDEPLOY_USER}/g",
                "s/^CONFIG_BUCKET=.*/CONFIG_BUCKET=${env.S3_CONFIGURATIONS_BUCKET}/",
                "s/^ECR_REPOSITORY_URI=.*/ECR_REPOSITORY_URI=${env.ECR_REPOSITORY_URI}/",
                "s/^GIT_COMMIT=.*/GIT_COMMIT=${revision}/",
                "s/^AWS_REGION=.*/AWS_REGION=${env.AWS_DEFAULT_REGION}/",
            ])
            sh "tar -cvzf frontend-renderer-${revision}.tar.gz appspec.yml scripts/codedeploy"
            sh "aws s3 cp frontend-renderer-${revision}.tar.gz s3://${env.S3_REVISIONS_BUCKET}/"
        }

        if (env.JOB_NAME.replaceFirst('.+/', '') != 'develop') return

        stage('Deploy') {
            for (group in [env.CODEDEPLOY_FRONTEND_DEPLOYMENT_GROUP, env.CODEDEPLOY_PUBLISHING_DEPLOYMENT_GROUP]) {
                sh sprintf('aws deploy create-deployment %s %s %s,bundleType=tgz,key=%s', [
                    '--application-name frontend-renderer',
                    "--deployment-group-name ${group}",
                    "--s3-location bucket=${env.S3_REVISIONS_BUCKET}",
                    "frontend-renderer-${revision}.tar.gz",
                ])
            }
        }
    }
}

@NonCPS
def revisionFrom(tag, commit) {
    def matcher = (tag =~ /^release\/(\d+\.\d+\.\d+(?:-rc\d+)?)$/)
    matcher.matches() ? matcher[0][1] : commit
}
