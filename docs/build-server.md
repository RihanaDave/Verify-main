<!--
GENERATED FILE - DO NOT EDIT
This file was generated by [MarkdownSnippets](https://github.com/SimonCropp/MarkdownSnippets).
Source File: /docs/mdsource/build-server.source.md
To change this file edit the source file and then run MarkdownSnippets.
-->

# Build Server


## Getting .received files from CI


### AppVeyor

Use a [on_failure build step](https://www.appveyor.com/docs/build-configuration/#build-pipeline) to call [Push-AppveyorArtifact](https://www.appveyor.com/docs/build-worker-api/#push-artifact).<!-- include: build-server-appveyor. path: /docs/mdsource/build-server-appveyor.include.md -->

```
on_failure:
  - ps: Get-ChildItem *.received.* -recurse | % { Push-AppveyorArtifact $_.FullName -FileName $_.Name }
```

See also [Pushing artifacts from scripts](https://www.appveyor.com/docs/packaging-artifacts/#pushing-artifacts-from-scripts).<!-- endInclude -->


### GitHub Actions

Use a [if: failure()](https://docs.github.com/en/free-pro-team@latest/actions/reference/context-and-expression-syntax-for-github-actions#failure) condition to upload any `*.received.*` files if the build fails.<!-- include: build-server-githubactions. path: /docs/mdsource/build-server-githubactions.include.md -->

```yaml
- name: Upload Test Results
  if: failure()
  uses: actions/upload-artifact@v2
  with:
    name: verify-test-results
    path: |
      **/*.received.*
```
<!-- endInclude -->


### Azure DevOps YAML Pipeline

Directly after the test runner step add a build step to set a flag if the testrunner failed. This is done by using a [failed condition](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/conditions?view=azure-devops&tabs=yaml). This flag will be evaluated in the CopyFiles and PublishBuildArtifacts steps below.<!-- include: build-server-azuredevops. path: /docs/mdsource/build-server-azuredevops.include.md -->

```yaml
- task: CmdLine@2
  displayName: 'Set flag to publish Verify *.received.* files when test step fails'
  condition: failed()
  inputs:
    script: 'echo ##vso[task.setvariable variable=publishverify]Yes'
```

Since the PublishBuildArtifacts step in DevOps does not allow a wildcard it is necessary to need stage the 'received' files before publishing:

```yaml
- task: CopyFiles@2
  condition: eq(variables['publishverify'], 'Yes')
  displayName: 'Copy Verify *.received.* files to Artifact Staging'
  inputs:
    contents: '**\*.received.*' 
    targetFolder: '$(Build.ArtifactStagingDirectory)\Verify'
    cleanTargetFolder: true
    overWrite: true
```

Publish the staged files as a build artifact:

```yaml
- task: PublishBuildArtifacts@1
  displayName: 'Publish Verify *.received.* files as Artifacts'
  name: 'verifypublish'
  condition: eq(variables['publishverify'], 'Yes')
  inputs:
    PathtoPublish: '$(Build.ArtifactStagingDirectory)\Verify'
    ArtifactName: 'Verify'
    publishLocation: 'Container'
```
<!-- endInclude -->


## Custom directory and file name

In some scenarios, as part of a build, the test assemblies are copied to a different directory or machine to be run. In this case custom code will be required to derive the path to the `.verified.` files. This can be done using [DerivePathInfo](naming.md#derivepathinfo).

For example a possible implementation for [AppVeyor](https://www.appveyor.com/) could be:

<!-- snippet: DerivePathInfoAppVeyor -->
<a id='snippet-derivepathinfoappveyor'></a>
```cs
if (BuildServerDetector.Detected)
{
    var buildDirectory = Environment.GetEnvironmentVariable("APPVEYOR_BUILD_FOLDER")!;
    DerivePathInfo(
        (sourceFile, projectDirectory, typeName, methodName) =>
        {
            var testDirectory = Path.GetDirectoryName(sourceFile)!;
            var testDirectorySuffix = testDirectory.Replace(projectDirectory, string.Empty);
            return new(directory: Path.Combine(buildDirectory, testDirectorySuffix));
        });
}
```
<sup><a href='/src/Verify.Tests/Snippets/Snippets.cs#L86-L100' title='Snippet source file'>snippet source</a> | <a href='#snippet-derivepathinfoappveyor' title='Start of snippet'>anchor</a></sup>
<!-- endSnippet -->