pipeline {
  agent {        
    dockerfile {
        filename 'Dockerfile.SecOps'
        // args '-v $HOME/.m2:/root/.m2'
        }
    }
  environment {
    // COP_SERVER_URL = credentials('server-url-cop')
    // COP_ACCESS_TOKEN = credentials('access-token-cop')
    // SRM_SERVER_URL = credentials('server-url-srm')
    // SRM_ACCESS_TOKEN = credentials('access-token-srm')
  }
  parameters {
      stashedFile 'source.zip'
      string(name: 'project_name', defaultValue: 'noname', description: '(Required) Provide a name for the project (no spaces in file-name)')
  }
  stages {
    stage('Initializa') {
      steps {
          cleanWs()
          script{
            unstash 'source.zip'
            sh '''
              set +x
              pwd && ls -la
              cat /etc/*release >> agent-info.md
              datetime=`date +"%Y-%m-%dT%H:%M:%SZ"`
              echo "$datetime" >> agent-info.md
              mkdir archive
              mv agent-info.md ./archive/agent-info.md

              if [ -z "${project_name}" ]; then
                echo "No project name provided. Using: no-name-$datetime"
                project_name="no-name-$datetime"
              else
                echo "Project Name: ${project_name}"
              fi
              echo "${project_name}">projectName.txt

              if [ -f "source.zip" ] && [ -s "source.zip" ]; then
                echo 'Source zip found!'
                echo false>skipScan.txt
                mkdir code
                unzip source.zip -d ./code/
              else
                echo 'No source zip found!'
                echo true>skipScan.txt
              fi
            '''
            script {
              def skipScanFile = readFile(file: "./skipScan.txt")
              def projectNameFile = readFile(file: "./projectName.txt")


              skipScanFile = skipScanFile.trim()
              projectNameFile = projectNameFile.trim()


              env.skipScanEnv = skipScanFile
              env.projectNameEnv = projectNameFile
            }
          }
      }
    }
    stage('CLoC') {
      when {
        expression {
          env.skipScanEnv == 'false'
        }
      }
      steps {
        sh '''
          set +x
          echo "Project: ${projectNameEnv}"
          echo "Running CLoC - v$(cloc.pl --version)"
          cloc.pl --md --out='./archive/cloc.md' ./code
        '''
      }
    }
    stage('Semgrep') {
      when {
        expression {
          env.skipScanEnv == 'false'
        }
      }
      steps {
        catchError(buildResult: 'FAILURE', message: 'Semgrep Scan Failed!', stageResult: 'FAILURE') {
          sh '''
            set +x
            pwd && ls -latr
            cd code/
            pwd && ls -latr
            echo "Running $(semgrep --version)"
            semgrep scan -o semgrep.json

            mv semgrep.json ../archive/
          '''
        }
      }
    }
    stage('Clean Up') {
      steps {
        sh '''
          set +x
          echo "Contents of $(pwd)..."
          ls -latr
          echo "Cleaning $(pwd)..."
          [ -f "skipScan.txt" ] && rm "skipScan.txt"
          [ -f "projectName.txt" ] && rm "projectName.txt"
          [ -f "source.zip" ] && rm source.zip
          [ -f ".polaris-coverity.zip" ] && rm .polaris-coverity.zip
          [ -d "code" ] && rm -rf code/
          ls -latr
        '''
      }
    }
  }
  post {
    always {
      archiveArtifacts artifacts: "**/*.md, *.md",
          allowEmptyArchive: true,
          fingerprint: true,
          onlyIfSuccessful: true
      cleanWs(cleanWhenNotBuilt: true,
          deleteDirs: true,
          disableDeferredWipeout: true,
          notFailBuild: true,
          patterns: [
            [pattern: '.gitignore', type: 'INCLUDE'],
            [pattern: '.propsfile', type: 'EXCLUDE']])
    }
  }
}