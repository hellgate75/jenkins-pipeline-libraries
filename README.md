# Jenkins Pipeline Libraries
[![Build Status](https://travis-ci.org/hellgate75/jenkins-pipeline-libraries.svg?branch=master)](https://travis-ci.org/hellgate75/jenkins-pipeline-libraries)

Useful Jenkins Pipeline Libraries to use for whatever.

## Prerequisites

* Jenkins 2.30+
* [Pipeline Shared Libraries](https://github.com/jenkinsci/workflow-cps-global-lib-plugin) plugin
* Other plugins may be required for specific library calls (i.e. Docker)

## Usage

1. Add global library 'pipelines' pointing to github in Jenkins settings (Manage Jenkins > Configure System > Global Pipeline Libraries)
2. Add `@Library('pipelines')` into your pipeline definition (more details [here](https://github.com/jenkinsci/workflow-cps-global-lib-plugin))

## Examples

### Build and push tag
```groovy
registry = '<SOME ECR REGISTRY>'

stage "Set Up"
node{
    sh("if find lib/*; then rm lib/*; fi && curl -L https://github.com/hellgate75/jenkins-pipeline-libraries/archive/${env.PIPELINE_LIBS_VERSION}.zip -o lib.zip && echo 'A' | unzip -j lib.zip */lib/* -d lib")

    shell = load "lib/shell.groovy"
    pom = load "lib/pom.groovy"
    ecr = load "lib/ecr.groovy"
    tools = load "lib/tools.groovy"
}

node{

    git(url: "https://github.com/dermotmburke/hello-boot.git")
    def version = pom.version("pom.xml")
    def artifactId = pom.artifactId("pom.xml")

    stage "Build"
    tools.configureMaven()
    sh("mvn clean package")

    docker.withRegistry(registry) {
        ecr.authenticate("us-east-1")

        def image = docker.build("${artifactId}:v${version}", '.')

        stage "Docker Push"
        image.push(version)
        image.push("latest")
    }
}
```

### Select tag
```groovy
registry = '<SOME ECR REGISTRY>'

stage "Set Up"
node{
    sh("if find lib/*; then rm lib/*; fi && curl -L https://github.com/hellgate75/jenkins-pipeline-libraries/archive/${env.PIPELINE_LIBS_VERSION}.zip -o lib.zip && echo 'A' | unzip -j lib.zip */lib/* -d lib")

    ui = load "lib/ui.groovy"
    ecr = load "lib/ecr.groovy"
    template = load "lib/template.groovy"
}

node{
    git(url: "https://github.com/dermotmburke/hello-boot.git")

    def tag = input(message: "Select Tag", parameters: [ui.dropdown("tag", "Tag") {
        ecr.imageTags("hello-boot", "us-east-1")
    }], submitter: null)

    def tmpFile = UUID.randomUUID().toString() + ".tmp"
    def ymlData = template.transform(readFile("docker-compose.yml.template"), [tag :tag])

    writeFile(file: tmpFile, text: ymlData)

    sh("convox login console.convox.com --password <PASSWORD>")
    sh("convox switch development")
    //sh("convox apps create hello-boot")
    sh("convox deploy --app hello-boot --file ${tmpFile}")

}
```
### License

[LGPL v.3](https://github.com/hellgate75/jenkins-pipeline-libraries/tree/master/LICENSE)
