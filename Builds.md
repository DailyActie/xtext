# The Build Infrastructure of Xtext

Xtext uses [Gradle](https://gradle.org) to build the Java projects that are independent of Eclipse, [Maven](https://maven.apache.org) to build the Maven plug-ins, and [Tycho](https://eclipse.org/tycho/) to build the Eclipse plug-ins. The builds are executed on a [Jenkins server](http://services.typefox.io/open-source/jenkins/).

An overview of the project dependencies and their build systems is shown here:
```
xtext-lib
(Gradle*)
    |
    |
xtext-core
(Gradle*)
    |
    |
xtext-extras________ ____________ ___________
 (Gradle*)          |            |           |
    |               |            |           |
    |               |            |           |
xtext-eclipse   xtext-idea   xtext-web   xtext-maven
 (Tycho)         (Gradle)    (Gradle)      (Maven)
    |
    |
xtext-xtend
(Gradle + Maven + Tycho)
    |
    |
xtext-umbrella
 (Tycho)
```

The Gradle projects marked with `*` include generated Tycho builds for creating p2 repositories as described below. For `xtext-xtend`, only the Tycho build depends on `xtext-eclipse`, while the Gradle and Maven builds just require `xtext-extras`.

## The Build Systems

### Gradle

The projects that include Gradle builds are `xtext-lib`, `xtext-core`, `xtext-extras`, `xtext-idea`, `xtext-web`, and `xtext-xtend`. These repositories share some common concepts:

* `settings.gradle` in the project root lists the subprojects to be included in the build.
* `build.gradle` in the project root applies commonly used plugin-ins such as `java` and applies configuration scripts from the `gradle` directory to all subprojects. The intent of each configuration script is explained in a file header comment.
* Each subproject has its own `build.gradle` that declares a title, description, and dependencies.
* The project version as well the versions of common dependencies are declared in `versions.gradle`.
* The remote repositories from which dependencies are drawn are declared in `upstream-repositories.gradle`. Depending on the `useJenkinsSnapshots` property, dependencies to upstream Xtext projects are resolved either against the Maven repositories created by [Jenkins build jobs](#the-build-servers) or against [public snapshots](https://oss.sonatype.org/content/repositories/snapshots).
* The [Xtend](http://xtend-lang.org) code is compiled with the [xtext-gradle-plugin](https://github.com/xtext/xtext-gradle-plugin). Since this plug-in has dependencies to projects that we want to compile, we have a bootstrap problem, which is solved using a shared dependency configuration declared in the root project (see `bootstrap-setup.gradle`). For `xtext-idea` and `xtext-web` the Java code generated by Xtend is not tracked in the git repository. For the other repositories the generated code is tracked in order to verify changes to the compiler. Here the compiler is activated with the `compileXtend` property.
* All build artifacts are installed into a local Maven repository `build/maven-repository` (see `maven-deployment.gradle`). The `pom.xml` files are generated with the `maven` Gradle plug-in.
* The Gradle projects `xtext-lib`, `xtext-core`, `xtext-extras`, and `xtext-xtend` are published both as Maven artifacts and as Eclipse plug-ins. Here the `MANIFEST.MF` files are maintained manually, which means they need to be kept consistent with the `build.gradle` configuration. The build timestamp is inserted into the version qualifier of these manifests automatically (see `manifest-gen.gradle`). For the generated source bundles, the manifest is completely generated from the project metadata. The `xtext-web` project is not published for Eclipse, but here we generate OSGi-compatible manifests using the `osgi` Gradle plug-in.
* The builds of `xtext-lib`, `xtext-core`, and `xtext-extras` include a Gradle plug-in that can generate a second Tycho-based build into the `releng` directory by running `./gradlew generateP2Build` (see `p2-deployment.gradle`). This Tycho build creates a local p2 repository in `build/p2-repository`. This approach eliminates the need to manually keep these Tycho builds consistent with the Gradle builds, e.g. regarding version numbers.
* The Xtext generator can be invoked from Gradle for the languages contained in the Xtext projects (see `mwe2-workflows.gradle`). For example, test languages can be generated with `./gradlew generateTestLanguages`.

### Maven Plug-ins

Maven plug-ins are built by the `xtext-maven` project and by the `org.eclipse.xtend.maven.*` subprojects in `xtext-xtend`.

* The root `pom.xml` (in `xtext-xtend` it's named `maven-pom.xml`) lists the subprojects to be included in the build.
* The parent modules `org.eclipse.xtext.maven.parent` and `org.eclipse.xtend.maven.parent` define the common plug-in configuration and metadata.
* Depending on the chosen profile, dependencies to upstream Xtext projects are resolved either against the Maven repositories created by [Jenkins build jobs](#the-build-servers) (`useJenkinsSnapshots` profile) or against [public snapshots](https://oss.sonatype.org/content/repositories/snapshots) (`useSonatypeSnapshots` profile).
* The `deploy` goal installs the artifacts into a local Maven repository `build/maven-repository`.

### Tycho

The Eclipse plug-ins and features of `xtext-eclipse` and `xtext-xtend` are built with Tycho.

* The root `pom.xml` (in `xtext-xtend` it's named `tycho-pom.xml`) lists the subprojects to be included in the build.
* The parent modules `org.eclipse.xtext.tycho.parent` and `org.eclipse.xtend.tycho.parent` define the common plug-in configuration.
* The target platform modules `org.eclipse.xtext.target` and `org.eclipse.xtend.target` define from where to fetch the dependencies. This includes external dependencies such as EMF as well as upstream Xtext projects, which are referenced via the p2 repositories created by [Jenkins build jobs](#the-build-servers).
* The modules `org.eclipse.xtext.p2-repository` and `org.eclipse.xtend.p2-repository` list the plug-ins and features to be exported. The resulting p2 repositories are copied to `build/p2-repository`.
* The [Xtend](http://xtend-lang.org) code is compiled with the [xtend-maven-plugin](https://github.com/eclipse/xtext-xtend/tree/master/org.eclipse.xtend.maven.plugin).

Another Tycho build is found in `xtext-umbrella`, with a very similar structure as described above. The difference is that the umbrella does not build its own plug-ins, but fetches all plug-ins from the other projects, builds some cross-project features (Xtext and Xtend SDKs), and exports them to a common p2 repository. The resulting p2 repository is used to create the Xtext update site for Eclipse.

## The Build Servers

### Jenkins

Each project has its own [multibranch pipeline](https://jenkins.io/blog/2015/12/03/pipeline-as-code-with-multibranch-workflows-in-jenkins/) on the [Jenkins build server](http://services.typefox.io/open-source/jenkins/). The server polls the git repositories every few minutes and automatically refreshes its build jobs: a new job is created for each new branch that is found, and the jobs of branches that do not exist anymore are deleted. This approach gives committers a very convenient way to test their changes on the server without affecting the main development streams (master and maintenance branches).

The actual build job description is written in a [Jenkinsfile](https://jenkins.io/doc/book/pipeline/jenkinsfile/). This file is included in the git repositories and thus can be modified per branch. Basically it defines the commands to execute for each build, where to find test results, and which artifacts to make available.

The build artifacts of each project are made available as a Maven repository, a p2 repository, or both. The build jobs of downstream projects are configured to consume the build artifacts of the projects they depend on. Gradle and Maven plug-in builds pick up their dependencies from the Maven repositories generated for their upstream projects, while Tycho builds use the p2 repositories.

### Hudson

The actual publishing is done on a [Hudson build server](https://hudson.eclipse.org/xtext/) by two build jobs: [xtext-snapshots](https://hudson.eclipse.org/xtext/job/xtext-snapshots/) for nightly snapshots and [xtext-release](https://hudson.eclipse.org/xtext/job/xtext-release/) for milestones and releases. Both employ the [publishing](https://github.com/TypeFox/publishing) project, which does the following:

1. Download Maven artifacts with a specified version from the repositories that are made available by the various Jenkins build jobs as described above.
2. [Sign the jar files](https://wiki.eclipse.org/JAR_Signing) with the remote service of the Eclipse Foundation.
3. Create additional signature files for publishing to remote Maven repositories.
4. Upload the results either to a snapshot repository or to a staging repository.
5. Download the common p2 repository from the Jenkins build job of `xtext-umbrella` as a zip file and unzip it.
6. Sign the jar files that have not already been signed in step 2 (results of the signing service are reused if the input files are equal).
7. Copy the signed jars and p2 repository metadata to the `build-result` directory and zip it.
8. Generate ant scripts for deploying the p2 repository to the Eclipse public download area (this might be improved in the future).

## The Release Process

#### Overview of the Process

1. Development phase
   1. Work on new features and issues.
   2. [Publish milestones](#preparing-milestones-and-releases) as required, e.g. when important new features have been implemented.
2. Release phase
   1. Stop implementing new features, but work on bug fixes.
   2. Create a release branch and [publish release candidate milestones](#preparing-milestones-and-releases) if required.
   3. [Lift the version number](#lifting-the-version-number) on the `master` branch and start the development phase for the next major/minor release (in parallel to the following steps).
   4. Create a test plan and test both old and new features.
   5. Prepare release notes.
   6. Merge or cherry-pick the bug fixes between the `master` branch and the release branch.
   7. [Publish the release](#preparing-milestones-and-releases).
   8. Create and push git tags.
   9. [Close the GitHub milestone](#github-milestones) and create new milestones.
3. Maintenance phase
   1. Work on bug fixes on the `master` branch.
   2. When it appears appropriate, create a service release branch and cherry-pick the bug fixes.
   3. [Publish a service release](#preparing-milestones-and-releases).
   4. Create and push git tags.
   5. If necessary, repeat the maintenance phase.

When major or minor releases are done, there should be a time period of at least one week when only important fixes are added to the code base to be released. This can be achieved by creating the release branch and committing the fixes directly to that branch. The release branch is merged back into `master` just before the actual release is done so the fixes are available there, too.

### Preparing Milestones and Releases

Branch names should be `milestone_«version»` for milestones, and `release_«version»` for releases. Tag names should be `v«version»`. When updating branch names for upstream dependencies, care must be taken to select the correct versions for additional libraries that are included in the Xtext build infrastructure (LSP4J).

The Xtend compiler version used in the build should be the current snapshot or the last milestone for milestones, and it should always be the last milestone for releases (this is important for making release builds reproducible).

Build jobs for releases must be executed in proper order on the build server, i.e. from upstream to downstream jobs. For every upstream change, the downstream jobs must be retriggered manually.

1. xtext-lib
   * Create release branch.
   * Set `version` property in `gradle/versions.gradle` to the release version.
   * Set `bootstrapXtendVersion` property in `gradle/bootstrap-setup.gradle` to the used Xtend compiler version.
   * `./gradlew generateP2Build`
2. xtext-core
   * Create release branch.
   * Set `version` property in `gradle/versions.gradle` to the release version.
   * Set `xtext_bootstrap` property in `gradle/versions.gradle` to the used Xtend compiler version.
   * `./gradlew generateP2Build -PuseJenkinsSnapshots=true`
3. xtext-extras
   * Create release branch.
   * Set `version` property in `gradle/versions.gradle` to the release version.
   * Set `xtext_bootstrap` property in `gradle/versions.gradle` to the used Xtend compiler version.
   * `./gradlew generateP2Build -PuseJenkinsSnapshots=true`
4. xtext-eclipse
   * Replace all occurrences of `job/master` according to the release branch name.
   * Set `xtend-maven-plugin-version` property in `releng/org.eclipse.xtext.tycho.parent/pom.xml` to the used Xtend compiler version.
5. xtext-idea
   * Create release branch.
   * Set `version` property in `gradle/versions.gradle` to the release version.
   * Set `bootstrapXtendVersion` property in `gradle/bootstrap-setup.gradle` to the used Xtend compiler version.
6. xtext-web
   * Create release branch.
   * Set `version` property in `gradle/versions.gradle` to the release version.
   * Set `bootstrapXtendVersion` property in `gradle/bootstrap-setup.gradle` to the used Xtend compiler version.
7. xtext-maven
   * Replace all occurrences of the previous version with the release version.
   * Replace all occurrences of `job/master` according to the release branch name.
8. xtext-xtend
   * Create release branch.
   * Set `version` property in `gradle/versions.gradle` to the release version.
   * Set `bootstrapXtendVersion` property in `gradle/bootstrap-setup.gradle` to the used Xtend compiler version.
   * Replace all occurrences of the previous version with the release version in the Maven plugin related pom.xml files (`maven-pom.xml`, `org.eclipse.xtend.maven.*`).
   * Replace all occurrences of `job/master` according to the release branch name.
   * Set `xtend-maven-plugin-version` property in `releng/org.eclipse.xtend.tycho.parent/pom.xml` to the used Xtend compiler version.
9. xtext-umbrella
   * Replace all occurrences of `job/master` according to the release branch name.
   * Update the name of the zipped p2 repository  according to the release version in `releng/org.eclipse.xtext.sdk.p2-repository/pom.xml` (`tofile` property).
10. Once all previous builds are successful, trigger the build job https://hudson.eclipse.org/xtext/job/xtext-release/ with the release version and branch name as parameters.
11. Ask @dhuebner to publish the Eclipse and Maven artifacts.

### Lifting the Version Number

Once the release branch for a major or minor release has been created, the master branch should be lifted to the next version.

Note that the Xtend compiler cannot be set to use snapshot versions from the beginning, since the new snapshots do not exist yet. It should be set to the latest published version (a release candidate or the actual release), and changed to the new snapshot version when it's available.

1. xtext-lib
   * Set `version` property in `gradle/versions.gradle` to the next version.
   * Set `bootstrapXtendVersion` property in `gradle/bootstrap-setup.gradle` to the used Xtend compiler version.
   * Replace occurrences of the previous version in `MANIFEST.MF` and `feature.xml` files.
   * `./gradlew generateP2Build`
2. xtext-core
   * Set `version` property in `gradle/versions.gradle` to the next version.
   * Set `bootstrapXtendVersion` property in `gradle/bootstrap-setup.gradle` to the used Xtend compiler version.
   * Replace occurrences of the previous version in `MANIFEST.MF` and `feature.xml` files.
   * `./gradlew generateP2Build -PuseJenkinsSnapshots=true`
3. xtext-extras
   * Set `version` property in `gradle/versions.gradle` to the next version.
   * Set `bootstrapXtendVersion` property in `gradle/bootstrap-setup.gradle` to the used Xtend compiler version.
   * Replace occurrences of the previous version in `MANIFEST.MF` and `feature.xml` files.
   * `./gradlew generateP2Build -PuseJenkinsSnapshots=true`
4. xtext-eclipse
   * Replace occurrences of the previous version in `MANIFEST.MF`, `feature.xml`, `pom.xml`, and `category.xml` files.
   * Set `xtend-maven-plugin-version` property in `releng/org.eclipse.xtext.tycho.parent/pom.xml` to the used Xtend compiler version.
5. xtext-idea
   * Set `version` property in `gradle/versions.gradle` to the next version.
   * Set `bootstrapXtendVersion` property in `gradle/bootstrap-setup.gradle` to the used Xtend compiler version.
6. xtext-web
   * Set `version` property in `gradle/versions.gradle` to the next version.
   * Set `bootstrapXtendVersion` property in `gradle/bootstrap-setup.gradle` to the used Xtend compiler version.
7. xtext-maven
   * Replace all occurrences of the previous version with the next version.
8. xtext-xtend
   * Replace all occurrences of the previous version with the next version.
   * Set `bootstrapXtendVersion` property in `gradle/bootstrap-setup.gradle` to the used Xtend compiler version.
   * Set `xtend-maven-plugin-version` property in `releng/org.eclipse.xtend.tycho.parent/pom.xml` to the used Xtend compiler version.
9. xtext-umbrella
   * Replace all occurrences of the previous version with the next version.

### GitHub Milestones

We use GitHub milestones to communicate when bug fixes and new features will be available and to generate a list of resolved issues for the release notes. When a major or minor release is done, the corresponding GitHub milestone should be closed. If there are open issues left in that milestone, they should be removed from it or assigned to another milestone.
