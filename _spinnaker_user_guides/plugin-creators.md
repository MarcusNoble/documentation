---
layout: post
title: Plugin Creators Guide
order: 108
---

Note that Plugins are currently in [Early Release](https://kb.armory.io/releases/early-release-beta-GA/). The feature is working and installable, but some functionality is likely to be missing and a number of known and unknown issues are likely to surface.

# Create The Backend For Stage Plugins
## Example Plugin

This document shows how to create a simple plugin that waits a random amount of time (from zero to the number of seconds that is entered in the UI). Use this guide as a starting point to facilitate creating more complex plugins. 

## Setting Up Your Project

To get started setting up your project, we highly suggest using [https://start.spring.io](https://start.spring.io/) to create your base project. 

Keep the following recommendations and requirements in mind:

- We recommend making the project a Gradle project. 
- You must set the base package path for the plugin to the following path: `com.netflix.spinnaker.plugin`. You can add anything else after that, but that is required. 
- You can add any dependencies you need to make your plugin successful.

Generate the project and unzip it to a location of your choosing. Modify the `build.gradle`  file to look like the following example:

```
plugins {
    id 'org.springframework.boot' version '2.1.8.RELEASE' apply false
    id 'io.spring.dependency-management' version '1.0.8.RELEASE'
    id 'java'
}

dependencyManagement {
    imports {
        mavenBom(org.springframework.boot.gradle.plugin.SpringBootPlugin.BOM_COORDINATES)
    }
}

group = 'com.netflix.spinnaker.plugin.armory' // Make sure this is your package path!
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '1.8'

repositories {
    mavenCentral()
    jcenter()
    maven { url "https://spinnaker.bintray.com/gradle" }
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    compile group: 'com.netflix.spinnaker.orca', name: 'orca-api', version: '7.36.0'
}
```

Then, we can remove both of the tests and the main application. (Don’t worry, we will add our own tests!)


## Creating the Plugin Stage

In order to create the stage plugin, we have to define three classes: Stage Input, Stage Output Context, and Stage Output Outputs. 

**SimpleStage Input**

SimpleStage Input is what our stage needs to use to do its job. The stage input comes from the Spinnaker UI. First, create a class that will be used as our Stage input. In our example, the plugin takes the max time to wait.

```
@Data
class RandomWaitInput {
    private int maxWaitTime;
}
```

**SimpleStage Context**

Context is used within the stage itself. In our example, add the maxWaitTime here.

```
@Data
class Context {
    private int maxWaitTime;
    public Context(int maxWaitTime) {
      this.maxWaitTime = maxWaitTime;
    }
}
```

**SimpleStage Output**

Output is what can be used later on in other stages. In our example, the output contains the actual number of seconds waited for the stage. 

```
@Data
class Output {
    private int timeToWait;
    public Output(int timeToWait) {
      this.timeToWait = timeToWait;
    }
}
```
## Create Stage Class

The stage itself needs to implement the [SimpleStage](https://github.com/spinnaker/orca/blob/ab89a0d7f847205ccd62e70f8a714040a8621ee7/orca-api/src/main/java/com/netflix/spinnaker/orca/api/SimpleStage.java) interface. The two methods that we need to implement are `getName` and `execute`.

**getName**

`getName` is a method that tells Spinnaker what the name of the stage is. 

**execute**

`execute` makes up the majority of the stage. `execute` takes in a `SimpleStageInput`, which then takes in (as a generic) the class that was created earlier for stage input. `execute`  returns a `SimpleStageOutput` that has our `Output` and `Context` classes. `SimpleStageOutput` also needs to know the status of the stage. This is where the [SimpleStageStatus](https://github.com/spinnaker/orca/blob/ab89a0d7f847205ccd62e70f8a714040a8621ee7/orca-api/src/main/java/com/netflix/spinnaker/orca/api/SimpleStageStatus.java) comes into play. Stages can be in the following states:

* Terminal → the stage failed
* Running → the stage is still executing
* Succeeded → the stage has successfully completed
* Not Started → the stage has not started yet

**Putting It All Together**

```
public class RandomWait<RandomWaitInput> {
  @Override
  public String getName() {
    return "randomWait";
  }

  @Override
  public SimpleStageOutput execute(SimpleStageInput<RandomWaitInput> stageInput) {
    Random rand = new Random();
    int maxWaitTime = stageInput.getValue().getMaxWaitTime();
    int timeToWait = rand.nextInt(maxWaitTime);

    try {
      TimeUnit.SECONDS.sleep(timeToWait);
    } catch(Exception e) {
      log.error("{}", e);
    }

    SimpleStageOutput<Output, Context> stageOutput = new SimpleStageOutput();
    Output output = new Output(timeToWait);
    Context context = new Context(maxWaitTime);

    stageOutput.setOutput(output);
    stageOutput.setContext(context);
    stageOutput.setStatus(SimpleStageStatus.SUCCEEDED);

    return stageOutput;
  }
}
```

# Create The Frontend For Stage Plugins

## Setting Up Your Project

### Rollup Configuration
Here is an example of a `rollup.config.js` to build your plugin:

```
import nodeResolve from 'rollup-plugin-node-resolve';
import commonjs from 'rollup-plugin-commonjs';
import typescript from 'rollup-plugin-typescript';
import postCss from 'rollup-plugin-postcss';
import externalGlobals from 'rollup-plugin-external-globals';

export default [
  {
    input: 'src/index.tsx',
    plugins: [
      nodeResolve(),
      commonjs(),
      typescript(),
      // Map imports from shared libraries (React, etc) to global variables exposed by Spinnaker.
      externalGlobals(spinnakerSharedLibraries()),
      // Import from .css, .less, and inject into the document <head></head>.
      postCss(),
    ],
    output: [{ dir: 'dist', format: 'es', }]
  }
];

function spinnakerSharedLibraries() {
  const libraries = ['react', 'react-dom', '@spinnaker/core'];

  function getGlobalVariable(libraryName) {
    const prefix = 'spinnaker.plugins.sharedLibraries';
    const sanitizedLibraryName = libraryName.replace(/[^a-zA-Z0-9_]/g, '_');
    return `${prefix}.${sanitizedLibraryName}`;
  }

  return libraries.reduce((globalsMap, libraryName) => {
    return { ...globalsMap, [ libraryName ]: getGlobalVariable(libraryName) }
  }, {});
}
```

`spinnakerSharedLibraries` pulls dependencies from Spinnaker. The libraries constant is a list of libraries that make the plugin work correctly. Items in this list must be from the [shared libraries](https://github.com/spinnaker/deck/blob/master/app/scripts/modules/core/src/plugins/sharedLibraries.ts#L32) exposed to plugin creators.

### Dependencies
As mentioned above, Spinnaker exposes libraries for plugins to use. Define dependencies in package.json. For this example plugin, the dependencies are:

```
"@spinnaker/core": "0.0.432",
"react": "^16.12.0",
"react-dom": "^16.12.0"
```

## Writing The Frontend

```
import * as React from 'react';
import { IStageTypeConfig, IStageConfigProps } from '@spinnaker/core';

const customStage: IStageTypeConfig = {
  label: 'Random Wait',
  description: 'Stage that waits a random amount of time up to the max inputted',
  key: 'randomWait',
  component: RandomWaitStage,
};

function setMaxWaitTime(event: React.SyntheticEvent, props: IStageConfigProps) {
  let target = event.target as HTMLInputElement;
  props.updateStageField({'maxWaitTime': target.value});
}

// Our stage component
function RandomWaitStage(props: IStageConfigProps) {
  return (
    <div>
      <label>
          Max Time To Wait
          <input value={props.stage.maxWaitTime} onChange={(e) => setMaxWaitTime(e, props)} id="maxWaitTime" />
      </label>
    </div>
  );
}

const plugin = {
  name: 'randomWait',
  stages: [customStage],
};

export { plugin };
```

### IStageTypeConfig
Define Spinnaker Stages with IStageTypeConfig. Required [options:](https://github.com/spinnaker/deck/blob/abac63ce5c88b809fcf5ed1509136fe96489a051/app/scripts/modules/core/src/domain/IStageTypeConfig.ts)
1. label -> The name of the Stage
2. description -> Long form that describes what the Stage actually does
3. key -> A unique name for the Stage
4. component -> The rendered React component

### IStageConfigProps
`IStageConfigProps` defines properties passed to all Spinnaker Stages. See [IStageConfigProps.ts](https://github.com/spinnaker/deck/blob/master/app/scripts/modules/core/src/pipeline/config/stages/common/IStageConfigProps.ts) for a complete list of properties. Pass a JSON object to the `updateStageField` method to add the `maxWaitTime` to the Stage.

### RandomWaitStage
This method returns [JSX](https://reactjs.org/docs/introducing-jsx.html) that gets displayed to the plugin user.

### How Spinnaker Loads The Plugin
Each plugin must export an object named `plugin`. You can only add Stages to this object. At startup, Spinnaker looks at `plugin.stages` and adds each defined Stage to the Stage Registry.

# Writing the Plugin Manifest

Here is an example of a plugin manifest:

```
name: armory/s3copy
description: Copies S3 files to different locations
manifestVersion: plugins/v1
version: 1.2.3
options:
  s3:
    username: user
    password: pass
resources:
  orca:
  - https://stage-plugin-test.s3-us-west-2.amazonaws.com/stage-plugin-0.0.1-SNAPSHOT.jar
  deck:
  - https://stage-plugin-test.s3-us-west-2.amazonaws.com/stage-plugin-ui-0.0.1-SNAPSHOT.js
```

The `name` is the name of the plugin. Names are namespaced so that plugins can have the same name but be by different vendors. In this example, the namespace is `armory` and the name of the plugin is `s3copy`.

The `description` gives the plugin user an idea of what the plugin does.

Plugin manifests can change overtime. The `manifestVersion` tells Spinnaker what version to use to validate the manifest. Currently, there is only the `plugins/v1` version.

`version` is the version of the plugin. 

Plugin users may want to change some settings to control how the plugin works. For example, controlling what username and password to use to connect to S3. The `options` key gives the plugin users that flexibility. Anything under `options` can be modified by the plugin user.

The next section in the manifest is for `resources`. Resources are things that are required for the plugin to run. For example, when creating a stage, there are jar(s) and Javascript code that need to be consumed by the plugin user. Currently, there are two different types of `resources`. The first is for Orca.  This is where we list a URL locations of where the jar(s) are for Orca to use. The second item under `resources` is for Deck, the frontend for Spinnaker. This is where a list of Javascript resources need to be put for Deck to consume.
