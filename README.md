# GITLAB + GITLAB RUNNER + ARTIFACTORY

**BUILDING INFRASTRUCTURE**

A painless and easiest way to set up an  infrastructure locally or on the remote machine using **Docker Compose**.

Create `docker-compose.yml` ****where you instruct docker ****to create  services:  
- Gitlab community edition,   
- Gitlab runner,   
- gitlab\_registration \( automated gitlab runner registration\) and finally   
- Jfrog Artifactory community edition 

In you working  directory where `docker-compose.yml` resides create folder **runner** and  place inside the `Dockerfile` and `runner_registration.sh` provided.  
The `Dockerfile` ****for the **runner\_registration** service simply runs the script and then exist. This registration service writes a config, then exits. The core settings of all is:

`gitlab_rails['initial_shared_runners_registration_token'] = "token"`

This allows us to know the token beforehand, without requiring the GUI.

The last thing required to make the above work is a suitable `.env` file in the same directory as your ****`docker-compose.yml` that sets all the relevant `${ENVIRONMENT_VARIABLE}` entries.

**USAGE**

`docker-compose up  -d`

and the when you run `docker-compose ps` you should see 3 containers for our services up and running:  
- gitlab container with the endpoint http://localhost:80  
- gitlab runner container  
- artifactory container with the endpoint http://localhost:81

**OUTCOME\_1**

You have  a fully healthy running environment to use Gitlab ,  Gitlab CI   and Jfrog ARTIFACTORY, now our task is to let Gitlab CI to automatically Build and Upload our binaries to ARTIFACTORY repository. Please see this below.

## DEPLOYING MAVEN PROJECT TO JFROG ARTIFACTORY WITH GITLAB CI/CD

**INTRO**

I've used an example on[`https://docs.gitlab.com/ee/ci/examples/artifactory_and_gitlab/`](https://docs.gitlab.com/ee/ci/examples/artifactory_and_gitlab/)  to deploy a maven project to Jfrog Artifactory by using a Gitlab CI/CD. We will create  a Maven application, our goal is to package and deploy the dependencies to Artifactory, to be available to other projects.

#### PREPARE THE DEPENDENCY APPLICATION <a id="prepare-the-dependency-application"></a>

1. Log in to your GitLab account
2. Create a new `your project` by selecting **Import project from ➔ Repo by URL**
3. Add the following URL: [`https://github.com/tidjumaev/petclinic-microservices-devops.git`](https://gitlab.com/gitlab-examples/maven/simple-maven-dep)
4. Click **Create project**

This application is nothing more than a basic class with a stub for a JUnit based test suite. It exposes a method called `hello` that accepts a string as input, and prints a hello message on the screen.

The project structure is really simple, and you should consider these two resources:

* `pom.xml`: project object model \(POM\) configuration file
* `src/main/java/com/example/dep/Dep.java`: source of our application

#### CONFIGURE THE ARTIFACTORY DEPLOYMENT <a id="configure-the-artifactory-deployment"></a>

The application is ready to use, but you need some additional steps to deploy it to Artifactory. There are two ways of doing so:

_**METHOD\_1:**_

1. Log in to Artifactory with your user’s credentials.
2. Create Maven repository `libs-release-local`
3. From the main screen, click on the `libs-release-local` item in the **Set Me Up** panel.
4. Copy to clipboard the configuration snippet under the **Deploy** paragraph.
5. Change the `url` value to have it configurable by using variables.
6. Copy the snippet in the `pom.xml` file for your project, just after the `dependencies` section. The snippet should look like this:

```text
<distributionManagement>
    <repository>
        <id>central</id>
        <name>9da9b1307b69-releases</name>
        <url>${env.MAVEN_REPO_URL}/libs-release-local</url>
    </repository>
</distributionManagement>
```

Another step you need to do before you can deploy the dependency to Artifactory is to configure the authentication data. It is a simple task, but Maven requires it to stay in a file called `settings.xml` that has to be in the `.m2` subdirectory in the user’s homedir.

1. Create a folder called `.m2` in the root of your repository
2. Create a file called `settings.xml` in the `.m2` folder
3. Copy the following content into a `settings.xml` file:

Since you want to use a runner to automatically deploy the application, you should create the file in the project’s home directory and set a command line parameter in `.gitlab-ci.yml` to use the custom location instead of the default one:

```text
<settings xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.1.0 http://maven.apache.org/xsd/settings-1.1.0.xsd"
    xmlns="http://maven.apache.org/SETTINGS/1.1.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <servers>
    <server>
      <id>central</id>
      <username>${env.MAVEN_REPO_USER}</username>
      <password>${env.MAVEN_REPO_PASS}</password>
    </server>
  </servers>
</settings>
```

Username and password will be replaced by the correct values using variables.

#### CONFIGURE GITLAB CI/CD FOR  **`your project`** <a id="configure-gitlab-cicd-for-simple-maven-dep"></a>

Now it’s time we set up [GitLab CI/CD](https://about.gitlab.com/stages-devops-lifecycle/continuous-integration/) to automatically build, test and deploy the dependency!

GitLab CI/CD uses a file in the root of the repository, named `.gitlab-ci.yml`, to read the definitions for jobs that will be executed by the configured runners. You can read more about this file in the [GitLab Documentation](https://docs.gitlab.com/ee/ci/yaml/README.html).

First of all, remember to set up variables for your deployment. Navigate to your project’s **Settings &gt; CI/CD &gt; Environment variables** page and add the following ones \(replace them with your current values, of course\):

* **MAVEN\_REPO\_URL**: `http://artifactory.example.com:8081/artifactory` \(your Artifactory URL\)
* **MAVEN\_REPO\_USER**: `gitlab` \(your Artifactory username\)
* **MAVEN\_REPO\_PASS**: `AKCp2WXr3G61Xjz1PLmYa3arm3yfBozPxSta4taP3SeNu2HPXYa7FhNYosnndFNNgoEds8BCS` \(your Artifactory Encrypted Password\)

Now it’s time to define jobs in `.gitlab-ci.yml` and push it to the repository:

```text
image: maven:latest

variables:
  MAVEN_CLI_OPTS: "-s .m2/settings.xml --batch-mode"
  MAVEN_OPTS: "-Dmaven.repo.local=.m2/repository"

cache:
  paths:
    - .m2/repository/
    - target/

build:
  stage: build
  script:
    - mvn $MAVEN_CLI_OPTS compile

test:
  stage: test
  script:
    - mvn $MAVEN_CLI_OPTS test

deploy:
  stage: deploy
  script:
    - mvn $MAVEN_CLI_OPTS deploy
  only:
    - master
```

The runner uses the latest [Maven Docker image](https://hub.docker.com/_/maven/), which contains all of the tools and dependencies needed to manage the project and to run the jobs.

Environment variables are set to instruct Maven to use the `homedir` of the repository instead of the user’s home when searching for configuration and dependencies.

Caching the `.m2/repository folder` \(where all the Maven files are stored\), and the `target` folder \(where our application will be created\), is useful for speeding up the process by running all Maven phases in a sequential order, therefore, executing `mvn test` will automatically run `mvn compile` if necessary.

Both `build` and `test` jobs leverage the `mvn` command to compile the application and to test it as defined in the test suite that is part of the application.

Deploy to Artifactory is done as defined by the variables we have just set up. The deployment occurs only if we’re pushing or merging to `master` branch, so that the development versions are tested but not published.

Done! Now you have all the changes in the GitLab repository, and a pipeline has already been started for this commit. In the **Pipelines** tab you can see what’s happening. If the deployment has been successful, the deploy job log will output:

```text
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 1.983 s
```

**Note**: the `mvn` command downloads a lot of files from the internet, so you’ll see a lot of extra activity in the log the first time you run it.

**OUTCOME\_2**

Checking in Artifactory will confirm that you have a new artifact available in the `libs-release-local` repository.  
