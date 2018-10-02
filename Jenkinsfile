#!groovy
node('maven') {
  slackSend channel: 'monolith', color: 'green', message: "started ${env.JOB_NAME} ${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)"
  sh "echo Hello"
  slackSend channel: 'monolith', color: 'green', message: "Successfully built ${env.JOB_NAME} ${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)"
}
