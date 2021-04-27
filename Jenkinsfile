import groovy.json.JsonOutput

node {
  def commitId, tag, commitDate

  stage('Prep') {
    checkout scm

    commitId = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
    commitDate =
        sh(script: "git show -s --format=%cd --date=format:%Y%m%d%H%M%S ${commitId}", returnStdout: true).trim()

    def pom = readMavenPom(file: 'pom.xml')
    def version
    if (pom.version.contains('-SNAPSHOT')) {
      version = pom.version.replace("-SNAPSHOT", "")
    }
    else {
      version = pom.version
    }
    version = version + "-${commitDate}-${commitId.substring(0, 7)}"
    tag = "depshield-$version"

    currentBuild.displayName = "#${currentBuild.number} - ${version}"
  }
  stage('Build') {
    withMaven(jdk: 'JDK8u172', maven: 'M3') {
      sh "mvn clean package"
    }
  }
  stage('Creating tag') {
    createTag nexusInstanceId: 'nxrm3', tagAttributesJson: "{'createdBy' : 'todo', 'hash' : '$commitId'}",
        tagName: "$tag"
  }
  stage('Publish') {
    // push jar
    nexusPublisher nexusInstanceId: 'nxrm3', nexusRepositoryId: 'test-maven-incoming',
        packages: [[$class: 'MavenPackage', mavenAssetList: [[ classifier: '', extension:
            '', filePath: 'target/my-app-1.0.jar']],
                    mavenCoordinate: [artifactId: 'my-app', groupId: 'com.mycompany.app',
                                      packaging : 'jar', version: '1.0']]],
        tagName: tag

    // push raw
    // note: would use nexusPublisher... but it doesn't support raw (or anything but Maven) ATM
    // note: would use httpRequest... but doesn't seem to have support for file upload
    def filename = "target/my-app-1.0-depshield.tar.gz"
    withMaven(jdk: 'JDK8u172', maven: 'M3', mavenSettingsConfig: 'nxrm3-server') {
       sh "mvn wagon:upload-single " +
           "-Dwagon.url=http://localhost:8081/repository/depshield-raw-incoming " +
           "-Dwagon.serverId=nxrm3-server " +
           "-Dwagon.fromFile=$filename " +
           "-Dwagon.toFile=my-app-1.0-depshield.tar.gz"
    }

    // tag raw
    sha1 = sha1 filename
    echo "SHA1 of '$filename': '$sha1'"

    // Can take a bit for the artifact to show up in search
    retry(10) {
      sleep 1
      // associate raw with tag
      def response = httpRequest url: "http://localhost:8081/service/rest/v1/tags/associate/${tag}?" +
            "repository=depshield-raw-incoming&format=raw&sha1=${sha1}&name=my-app-1.0-depshield.tar.gz",
            authentication: 'nxrm3-credentials',
            httpMode: 'POST',
            validResponseCodes: '200',
            acceptType: 'APPLICATION_JSON',
            contentType: 'APPLICATION_JSON'
        println("Status: " + response.status)
        println("Content: " + response.content)
    }

    def map = [tag: tag, sha1: sha1]
    def json = JsonOutput.toJson(map)
    //writeJSON file: 'build-info.json', json: json
    writeFile(file: 'build-info.json', text: json)
    archiveArtifacts artifacts: 'build-info.json', onlyIfSuccessful: true
  }
}
