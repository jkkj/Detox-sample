
# Building React Native app using Detox for testing

## Create React Native app

following [basic set up](https://reactnative.dev/docs/environment-setup) without expo

### Setup the environment

- Installing Node 14 or newer  
`curl -sL https://deb.nodesource.com/setup_14.x | sudo -E bash -`  
`sudo apt-get install -y nodejs`

- Installing Java Development Kit  
(React Native currently recommends version 11 of the JDK)  
for multiple jdk versions - what can be necessary since the version 11 is not the newest one - you can use `update-alternatives`  
Java can be installed on Ubuntu by the command  
`apt get install openjdk-11-jdk`  
then it could be set using update-alternatives like this  
`sudo update-alternatives --config java`  
where you choose the java 11 by the number

- Download Android studio  
  - there is [how to](https://developer.android.com/studio/intro#linux) on the official web page  
  - or you can use an [unofficial repo](https://github.com/mfonville/android-studio) and follow [these steps](https://vitux.com/how-to-install-android-studio-on-ubuntu/) - this could be more comfortable

- Install the Android SDK  
(Android 13 (Tiramisu) is required for React Native build)  
Can be installed through SDK manager in Android Studio

- Set ANDROID_HOME environment variable
- install Watchman - not necessary, highly recommended
- React Native Command Line Interface
- install detox command line tools  
`npm install detox-cli --global` maybe you would need to use `sudo` as well

### Creating a new application

- remove a global `react-native-cli` package if installed to avoid unexpected issues
- run  
`npx react-native@0.70.7 init AwesomeProject`  
(we used the version 0.70.7, it is the highest officially supported by Detox for now),  
replace `Awesome Project` by your preferred project name

### Run the app

- we can run the app on either physical device connected to the PC by USB or on an Android Virtual Device (AVD) which we need to create in Android Studio before
- first we need to start Metro - the javascript bundle shipped with React Native  
`npx react-native start`
- now, we can run the app by  
`npx react-native run-android`

## Add Detox

go to the project folder and all next steps do there (e.g. `cd AwesomeProject`)

- install `jest` (according to Detox documentation the most popular testing framework for React Native)  
`npm install "jest@^29" --save-dev`
- install detox itself  
`npm install detox --save-dev`
- initialize detox  
`detox init`  
(if you did not install detox command line tools than run `npx detox inti`)  
we should see the output like this  
`Create a file at path: .detoxrc.js`  
`Create a file at path: e2e/jest.config.js`  
`Create a file at paht: e2e/starter.test.js`  
- check the build path - if you use classic React Native app the config should be good, but for Expo or an app with flavours you will probably need to adjust `binaryPath` and `build` entries in `.detoxrc.js` file

## Device config

- in the file `detoxrc.js` fill the avdName by an actual name of a configured avd  
the list of avds could be obtained by `emulator -list-avds`

## Prepare & Run the first test

the above steps were not enough to run Detox on an android emulator, so we had to do some more Android specific steps:  

### Patching build scripts

- `android/build.gradle`:  

``` gradle
buildscript {
  ext {
    ...
    kotlinVersion = "1.7.20"
  }
  ...
  dependencies {
    ...
    classpath("org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlinVersion")
  }
}
...
allprojects {
    repositories {
        maven {
            url("$rootDir/../node_modules/detox/Detox-android")
        }
    }
}
```
where `1.7.20 ` is the current kotlin version

- `android/app/build.gradle`:  

``` gradle
...
android {
  ...
  defaultConfig {
    ...
    testBuildType System.getProperty('testBuildType', 'debug')
    testInstrumentationRunner 'androidx.test.runner.AndroidJunitRunner'
    ...
    }
    buildTypes {
      release {
        ...
        proguardFile "${rootProject.projectDir}/../node_modules/detox/android/detox/proguard-rules-app.pro"
      }
    }
    ...
    dependencies {
      androidTestImplementation('com.wix:detox:+')
      implementation 'androidx.appcompat:appcompat:1.1.0'
      ...
    }
}
```

- add auxiliary android test  
on the path `android/app/src/androidTest/java/com/<your.package>/DetoxTest.java`

```java
package com.<your.package>; // (1)

import com.wix.detox.Detox;
import com.wix.detox.config.DetoxConfig;

import org.junit.Rule;
import org.junit.Test;
import org.junit.runner.RunWith;

import androidx.test.ext.junit.runners.AndroidJUnit4;
import androidx.test.filters.LargeTest;
import androidx.test.rule.ActivityTestRule;

@RunWith(AndroidJUnit4.class)
@LargeTest
public class DetoxTest {
    @Rule // (2)
    public ActivityTestRule<MainActivity> mActivityRule = new ActivityTestRule<>(MainActivity.class, false, false); // 18

    @Test
    public void runDetoxTests() {
        DetoxConfig detoxConfig = new DetoxConfig();
        detoxConfig.idlePolicyConfig.masterTimeoutSec = 90;
        detoxConfig.idlePolicyConfig.idleResourceTimeoutSec = 60;
        detoxConfig.rnContextLoadTimeoutSec = (BuildConfig.DEBUG ? 180 : 60);

        Detox.runTests(mActivityRule, detoxConfig);
    }
}
```  

lines 1 and 18 need to be changed according to the project (the package name and the main activity class name)

- enabling unencrypted traffic for Detox  
`android/app/src/main/res/xml/network_security_config.xml`  

``` xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <domain-config cleartextTrafficPermitted="true">
        <domain includeSubdomains="true">10.0.2.2</domain>
        <domain includeSubdomains="true">localhost</domain>
    </domain-config>
</network-security-config>
```

`android/app/src/main/AndroidManifest.xml`

```xml
<manifest>
  <application
  ...
  android:networkSecurityConfig="@xml/network_security_config"
  >
  ...
  </application>
</manifest>
```

## Build the app

`detox build --configuration android.emu.debug`  
if you did not install detox cli, you could try to run it with the prefix `npx`  

## Run the tests

the test is for now not working properly and will 99% fail but you can run it by calling:  
`npx react-native start` or `npm start` if it is configured properly  
now you can run the tests by:  
`detox test --configuration android.emu.debug`  

there will be probably an error similar to this:

```console
 FAIL  e2e/starter.test.js (25.916 s)
  Example
    ✕ should have welcome screen (662 ms)
    ✕ should show hello screen after tap (236 ms)
    ✕ should show world screen after tap (236 ms)

  ● Example › should have welcome screen

    Test Failed: No elements found for “MATCHER(id == “welcome”)”

    HINT: To print view hierarchy on failed actions/matches, use log-level verbose or higher.

       9 |
      10 |   it('should have welcome screen', async () => {
    > 11 |     await expect(element(by.id('welcome'))).toBeVisible();
         |                                             ^
      12 |   });
      13 |
      14 |   it('should show hello screen after tap', async () => {

      at Object.toBeVisible (e2e/starter.test.js:11:45)

  …
```

$\color{red}most~problems~I~experienced~was~caused~by~wrong~cases~of~letters~and~quotes~vs~double~quotes$