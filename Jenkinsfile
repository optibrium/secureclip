node('docker') {

    checkout scm

    stage('Collect environment variables') {

        withCredentials([
            string(credentialsId: 'pypi-password', variable: 'PYPI_PASSWORD'),
            string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')
        ]) {
            env.TWINE_PASSWORD = "${PYPI_PASSWORD}"
            env.GITHUB_TOKEN = "${GITHUB_TOKEN}"

            env.PROJECT = 'optibrium/secureclip'
            env.BUILDCONTAINER = 'optibrium/buildcontainer:0.21.1'

            if (env.TAG_NAME ==~ /v[0-9]{1,}\.[0-9]{1,}\.[0-9]{1,}(\-.*)?/) {
                env.GIT_TAG = env.TAG_NAME.replaceFirst('v', '')
            }
        }
    }

    try {

        docker.image(env.BUILDCONTAINER).inside {

            stage('Build wheel') {
                script {
                    sh 'python3 setup.py bdist_wheel'
                }
            }

            if (env.GIT_TAG) {

                stage('Upload to PyPi') {

                    sh 'twine upload --repository-url https://pypi.infra.optibrium.com -u twine dist/*.whl'
                }
            }
        }

        if (env.GIT_TAG) {

            stage('build Docker image') {
                app = docker.build("optibrium/secureclip")
            }

            stage('tag Docker image') {
                app.tag(env.GIT_TAG)
            }

            stage('push Docker image') {
                app.push()
                app.push(env.GIT_TAG)
            }

            node('master') {
                stage('Update clip deployment in infra') {
                    sh "kubectl set image deployment/clip clip=optibrium/secureclip:${env.GIT_TAG}"
                }
            }
        }

        docker.image(env.BUILDCONTAINER).inside {

            stage('Report Success to Github') {
                sh 'report-to-github success'
            }
        }

    } catch (Exception failure) {

        docker.image(env.BUILDCONTAINER).inside {

            stage('Report Failure to Github') {
                sh 'report-to-github failure'
            }
        }

        throw failure
    }
}
