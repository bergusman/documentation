pipeline {
  agent any
  stages {
    stage('Build') {
      steps {
        sh '/usr/local/bin/bundle install && /usr/local/bin/bundle exec jekyll build'
      }
    }
  }
}
