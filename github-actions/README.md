# Getting Started With GitHub Actions and GitHub Packages

This tutorial introduces the basic features of GitHub Actions and GitHub Packages.

Given a Java project managed by Maven, we want to use GitHub Actions to run the tests, build the Docker image, and eventually push the image to GitHub Packages waiting for deployment.

## Prerequisites

You should be familiar with the basic commands of Docker.

## Continuous Integration with GitHub Actions

At this step, we first create a workflow from the default [Maven workflow template](https://github.com/actions/starter-workflows/blob/master/ci/maven.yml), generated in `.github/workflow`.
The detailed configuration is introduced in [Building and testing Java with Maven](https://docs.github.com/en/actions/language-and-framework-guides/building-and-testing-java-with-maven).

```yaml
name: Java CI with Maven

on:
  push:
    branches: [ $default-branch ]
  pull_request:
    branches: [ $default-branch ]

jobs:
  build:

    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v2
    - name: Set up JDK 1.8
      uses: actions/setup-java@v1
      with:
        java-version: 1.8
    - name: Build with Maven
      run: mvn -B package --file pom.xml
```

The above workflow defines three steps:

1. Checkout the repository.
2. Set up Java environment with JDK 1.8.
3. Build the package with Maven from `pom.xml`.

You definitely do not want to publish the code without unit testing.
As a result, we should run the tests before building the package, which can be simply implemented by adding the following two lines.

```diff
jobs:
  build:

    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v2
    - name: Set up JDK 1.8
      uses: actions/setup-java@v1
      with:
        java-version: 1.8
+   - name: Test with Maven
+     run: mvn test
    - name: Build with Maven
      run: mvn -B package --file pom.xml
```

Now the test should be run before the building process.
Finally, we want the workflow to be executed when we push to the `master` branch.

```yaml
name: Staging

on:
  push:
    branches: [ master ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - name: Set up JDK 1.8
      uses: actions/setup-java@v1
      with:
        java-version: 1.8
    - name: Test with Maven
      run: mvn test
    - name: Build with Maven
      run: mvn -B package --file pom.xml
```

## Publishing Docker Images to GitHub Packages

At this step, we want to use Docker to generate a image after the package is built.
If you are not quite familiar with Docker, you can visit [Docker Documentation](https://docs.docker.com/) to learn about the basic concepts of Docker.
A sample Dockerfile is defined below.
We simply copy the build JAR package to the working directory and use `java -jar` to run the package.
Notice that the port 8080 is exposed.

```dockerfile
FROM openjdk:15-jdk-alpine
WORKDIR /app
COPY target/*.jar /app
ENTRYPOINT ["java", "-jar", "demo-0.0.1-SNAPSHOT.jar"]
EXPOSE 8080
```

To push to the GitHub Packages, you will need to first generate a **personal access token** from `Settings > Developer settings > Personal access tokens > Generate new token` at GitHub.
Scopes defining the access for personal tokens should at least include `repo`, `write:packages` and `read:packages`.
The detailed instructions is introduced in [Authenticating with the GITHUB_TOKEN](https://docs.github.com/en/actions/configuring-and-managing-workflows/authenticating-with-the-github_token).

We cannot use the personal access token without encryption in the workflow file, so we have to put it in the `Secrets` of the repository. The variables defined in `Secrets` can be accessed by `${{ secrets.VARIABLE }}`.
Now we are ready to define the publishing workflow.

First, we want the workflow to be triggered when we make a release and build the Docker image from the built package. You can modify the event trigger and let the workflow be trigger when publishing a release.
Then we use the basic action `docker/build-push-action@v1`.
This action detects the Dockerfile defined in the repository.
Finally, the image is built from the Dockerfile and pushed to the GitHub Packages.

```yaml
name: Release

on:
  release:
    types: [published]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - name: Set up JDK 1.8
      uses: actions/setup-java@v1
      with:
        java-version: 1.8
    - name: Test with Maven
      run: mvn test
    - name: Build with Maven
      run: mvn -B package --file pom.xml
    - name: Push to GitHub Packages
      uses: docker/build-push-action@v1
      with:
        username: ${{ github.actor }}
        password: ${{ secrets.PERSONAL_ACCESS_TOKEN }}
        registry: docker.pkg.github.com
        repository: OWNER/REPOSITORY/IMAGE_NAME
        tag_with_ref: true
```

## Running the Web Service

You can run your web service from the Docker image pushed to GitHub Packages.

```shell
$ docker login https://docker.pkg.github.com -u USERNAME
Password: GITHUB_ACCESS_TOKEN
Login Succeeded

$ docker run -p PORT:8080 docker.pkg.github.com/OWNER/REPOSITORY/IMAGE_NAME:TAG_NAME
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.3.1.RELEASE)
```

## Summary

- Staging (push)
  1. Checkout the repository.
  2. Set up Java environment with JDK 1.8.
  3. Run unit tests.
  4. Build the package with Maven from pom.xml.
- Release (release)
  1. Checkout the repository.
  2. Set up Java environment with JDK 1.8.
  3. Run unit tests.
  4. Build the package with Maven from pom.xml.
  5. Build the Docker image from `Dockerfile` and publish it to GitHub Packages.

## Demo

Please visit [sjtu-se128/github-actions-java-demo](https://github.com/sjtu-se128/github-actions-java-demo) for more details.

## References

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [GitHub Packages Documentation](https://docs.github.com/en/packages)
- [Using GitHub Packages with GitHub Actions](https://docs.github.com/en/packages/using-github-packages-with-your-projects-ecosystem/using-github-packages-with-github-actions)
- [Publishing Docker images](https://docs.github.com/en/actions/language-and-framework-guides/publishing-docker-images)
- [Configuring Docker for use with GitHub Packages](https://docs.github.com/en/packages/using-github-packages-with-your-projects-ecosystem/configuring-docker-for-use-with-github-packages)
