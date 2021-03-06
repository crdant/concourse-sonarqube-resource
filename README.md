# SonarQube Resource

Performs SonarQube analyses and tracks the state of SonarQube quality gates.

## Requirements
* A running SonarQube instance (this resource was tested on v6.5, but it should
  work with every version of SonarQube >= 5.3)

## Installation
Add a new resource type to your Concourse CI pipeline:
```yaml
 resource_types:
 - name: sonar-runner
  type: docker-image
  source:
    repository: cathive/concourse-sonarqube-resource
    tag: latest # For reproducible builds use a specific tag and don't rely on "latest".
```

## Source Configuration

* `host_url`: *Required.* The address of the SonarQube instance,
  e.g. "https://sonarqube.example.com/". Must end with a slash.

* `login`: *Required.* The login or authentication token of a SonarQube user with Execute Analysis
  permission.

* `password`: The password that goes with the sonar.login username. This should be left blank if an
  authentication token is being used.

* `maven_settings`: Maven settings to be used when performing SonarQube analysis.
  Only used if the scanner_type during the out phase has been set to / determined to use
  Maven.

## Behavior

The resource implements all three actions (check, in and out).
The analysis is triggered by the out action and check/in will be used to wait for
the result of the analysis and pull in the project status. Tasks can use this
information to break the build (if desired) if any of the criterias of the
quality gate associated with a project are not met.

### out: Trigger SonarQube analysis

#### Parameters
* `project_path`: *Required* Path to the resource that shall be analyzed.
  If the path contains a file called "sonar-project.properties" it will be picked
  up during analysis.
* `scanner_type`: Type of scanner to be used. Possible values are:
  * `auto` (default) Uses the maven-Scanner if a pom.xml is found in the directory
    specified by sources, cli otherwise.
  * `cli` Forces usage of the command line scanner, even if a Maven project object
    model (pom.xml) is found in the sources directory.
  * `maven` Forces usage of the Maven plugin to perform the scan.
* `project_key`: Project key (default value is read from sonar-project.properties)
* `project_name`: Project name (default value is read from sonar-project.properties)
* `project_description`: Project description (default value is read from sonar-project.properties)
* `project_version`: Project version (default value is read from sonar-project.properties)
* `project_version_file`: File to be used to read the Project version. When this option has been specified, it has precedence over the project_version parameter.
* `branch`: SCM branch. Two branches of the same project are considered to be different projects in SonarQube. Therefore, the default SonarQube behavior is to set the branch to an empty string.
* `analysis_mode`: One of
  * `publish` - this is the default. It tells SonarQube to perform a full, store-it-in-the-database analysis.
  * `preview` - Currently not supported by this resource!
  * `issues` - Currently not supported by this resource!
* `sources`: Comma-separated paths to directories containing source files.
* `maven_settings_file`: Path to a Maven settings file that shall be used.
  Only used if the scanner_type during has been set to / determined to use Maven.
  If the resource itself has a maven_settings configuration, this key will override
  it's value.

### in: Fetch result of SonarQube analysis

The action will place two JSON files into the resource's folder which are fetched from
the SonarQube Web API:
* qualitygate_project_status.json
  Quality gate status of the compute engine task that was triggered by the resource
  during the out action.
  Format: https://next.sonarqube.com/sonarqube/web_api/api/qualitygates/project_status
* ce_task_info.json
  Information about the compute engine task that performed the analysis.
  Format: https://next.sonarqube.com/sonarqube/web_api/api/ce/task

## Full example

The following example pipeline shows how to use the resource to break the build if
a project doesn't meet the requirements of the associated quality gate.

 ```yaml
resource_types:
- name: sonar-runner
  type: docker-image
  source:
    repository: cathive/concourse-sonarqube-resource
    tag: latest # For reproducible builds use a specific tag and don't rely on "latest".

resources:
- name: example-sources-to-be-analyzed
  type: git
  source:
    uri: https://github.com/example/example.git
- name: code-analysis
  type: sonar-runner
  source:
    host_url: https://sonarqube.example.com/
    login: ((SONARQUBE_AUTH_TOKEN))
    project_key: com.example.my_project
    branch: master

jobs:
- name: build-and-analyze
  plan:
  - get: example-sources-to-be-analyzed
    trigger: true
  - task: build
    config:
      platform: linux
      image_resource:
        type: docker-image
        source:
          repository: debian
          tag: 'jessie'
        inputs:
        - name: example-sources-to-be-analyzed
        run:
          path: build.sh
          dir: example-sources-to-be-analyzed
  - put: sonar-runner
    params:
      project_path: example-sources-to-be-analyzed
- name: qualitygate
  plan:
  - get: sonar-runner
    passed:
    - build-and-analyze
    trigger: true
  - task: break-build
    config:
    platform: linux
    image_resource:
      type: docker-image
      source:
        repository: node
        tag: '8.4.0'
      inputs:
      - name: sonar-runner
      run:
        path: node
        args:
        - -e
        - "const projectStatus = require('./qualitygate_project_status.json'); if (projectStatus.status !== 'OK') { console.error('Quality gate goals missed. :-('); process.exit(1); }"
        dir: sonar-runner
```