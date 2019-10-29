# [fit] Adding more native to

# [fit] your React Native app

---

### Who are we?

- Eloy Durán
- @alloy
- Author of CocoaPods
- Director of Engineering at Artsy

^ Where we’ve been using React Native for 3.5 years in our brownfield app

---

### Who are we?

- Bartol Karuza
- @bartolkaruza
- Engineering Manager at Chargetrip
- React Native advocate and contributor

---

### What will you learn from this presentation?

- What native package ‘linking’ is
- How to use it for iOS and Android platforms

---

## [fit] What is it and what

## [fit] problems does it solve?

^ First, let’s talk about ‘linking’ _outside_ of the context of React Native.

^ Perhaps you may have thought it was related to `npm link` / `yarn link`; however, beyond both making dependencies whose source exists on a local file-system available to a project, these tools are meant to allow one to _work_ on those dependencies and take their name from ‘symbolic link’, the manner in which they are made available to an application. In case you were not thinking this–never mind me, let’s carry on.

^ Instead, the origins lie with tooling thats exist in native tool-chains, such as those used for languages like `C`, the term ‘linking’ refers to building up a working executable from various compiled modules–referred to as ‘object files’, that contain the compiled ‘object code’ corresponding to input source files. These can either be ‘statically’ linked, which basically just copies over the ‘object code’ into the executable; or they can be ‘dynamically’ linked, meaning the ‘object code’ is provided as a separate binary to the executable and loaded into memory at runtime. Why you would choose one over the other is beyond the scope of this presentation, but suffice it to say that for a native dependency’s code to be available to the application it needs to be linked in either of these manners.

^ ‘Linking’ in the context of React Native _refers_ to this process, but in reality only covers _configuring_ your project for the actual linker machinery to be able to do its work at build time. This is the part that the rest of this presentation is concerned about.

---

## [fit] How was this done

## [fit] historically (for iOS)?

^ Now that we understand what linking refers to in the context of React Native, let’s take a look at how this was done over the years in an iOS project.

---

### Manual

https://facebook.github.io/react-native/docs/linking-libraries-ios#manual-linking

^ As taken from the React Native documentation, a native package is expected to come with a Xcode project that instructs Xcode how to build a static library out of the package’s native source.

---

### Manual

1. Add to project

^ The user adds a reference to the package’s Xcode project to their own (app) Xcode project;

---

### Manual

1. Add to project
2. Setup linkage

^ after which they are able to select the package’s static library from the list of libraries their application should link against;

---

### Manual

1. Add to project
2. Setup linkage
3. Build settings

^ and finally, where necessary, update build settings of the app Xcode project required by the dependency, such as including the correct C++ standard library.

---

### Manual

1. Add to project
2. Setup linkage
3. Build settings
4. Setup for native code

^ If, at runtime, you are only using the package through JavaScript then you are done now. If you also need to use the package’s code from your own native code, you will need to setup header search paths so you can include the package’s headers in your source files.

---

### Manual

1. Add project
2. Setup linkage
3. Configure build settings
4. Setup for use in native code
5. Repeat for transitive dependencies

^ If the package has any native dependencies of its own, you need to repeat the above steps for those dependencies.

---

### Automated

`react-native link`

^ An automated version of the exact same approach has existed for a while now, through the `link` command of the `react-native` command-line tool, in that it will actually edit your Xcode project on-disk. One limitation with this approach was that the tool doesn’t know about **native** transitive dependencies, and as such in those cases you would still have to perform the manual approach for those dependencies.

---

### What problems exist with this approach?

1. Number of steps

^ In an ideal scenario, you would have to perform 2 steps to manually integrate a single native package, at worst including additional build settings and multiplied by the number of **native** transitive dependencies the native package has. Even in the automated case, the inconsistency in handling **native** transitive dependencies isn’t the greatest developer-experience.

---

### What problems exist with this approach?

1. Number of steps
2. Required knowledge

^ Needing to understand the build settings of Xcode and its underlying tooling. This isn’t necessarily a bad thing, things _can_ and _will_ break and it’s good to know how things work under the hood, but _having_ to do this each time even when on the happy path isn’t ideal.

---

### What problems exist with this approach?

1. Number of steps
2. Required knowledge
3. Maintenance burden

^ Other than bookkeeping, there’s no easy way to track which build settings originated from what dependency, which becomes a maintenance burden when you want to upgrade or remove dependencies.

---

## [fit] Alternative and the future

## [fit] of React Native linking

^ This is where relying on dependency managers comes in. Respective ecosystems already have dependency management solutions that are able to deal with all of the above, so in the spirit of standing on the shoulders of giants, let’s leverage those domain specific tools instead!

^ In the case of iOS, the popular option is [CocoaPods](https://cocoapods.org)–which given a manifest that describes a project’s dependencies, similarly to a `package.json` file, will perform all of the aforementioned work _and_ do so in a maintainable manner. For instance, it is able to resolve and install transitive dependencies like any other dependency, it encapsulates dependency specific build settings in `xcconfig` files, and does so with minimal _one-time_ changes to your app’s Xcode project.

^ Seeing as sooner or later most people will have to deal with linking native packages and the different ways in which you had to maintain your app’s Xcode project, the React Native maintainers decided to make utilizing CocoaPods the default since [version 0.60](https://facebook.github.io/react-native/blog/2019/07/03/version-60#cocoapods-by-default). (Gradle, for Android apps, was already the default.)

^ So, great–now there’s a single place where you manage your native dependencies! But given this standardization, we can take it one step further and automate the process. 🐢s all the way down!

---

### Auto-linking

^ Enter auto-linking, which is a new addition to the `react-native` command-line tool that discovers packages that have native requirements and offloads the responsibility of configuring the app project to the dependency manager for the respective platform.

^ For iOS, this means that adding a native package boils down to:

---

### Auto-linking

1. Install the package
1. Run `pod install`
1. There’s no step 3

^ What this does under the hood, is that the CocoaPods manifest, the `Podfile`, includes a React Native specific helper, which on `pod install` will ask the `react-native` command-line tool for a list of all the native packages and their corresponding `podspec` files–which are the files that describe to CocoaPods how to build the library. It then **dynamically** includes those dependencies in its set of dependencies to resolve, as if they were all explicitly specified in the manifest.

---

### Auto-linking

```ruby
require_relative "../node_modules/@react-native-community/cli-platform-ios/native_modules"
```

^ For instance, given a `Podfile` that includes the helper like so:

---

### Auto-linking

```
$ npx --quiet react-native config
```

^ …and with `lottie-react-native` as an example, this is what the helper will invoke:

---

### Auto-linking

```json
{
  "dependencies": {
    "lottie-react-native": {
      "root": "/path/to/project/node_modules/lottie-react-native",
      "name": "lottie-react-native",
      "platforms": {
        "ios": {
          "sourceDir": "/path/to/project/node_modules/lottie-react-native/src/ios",
          "folder": "/path/to/project/node_modules/lottie-react-native",
          "pbxprojPath": "/path/to/project/node_modules/lottie-react-native/src/ios/LottieReactNative.xcodeproj/project.pbxproj",
          "podfile": null,
          "podspecPath": "/path/to/project/node_modules/lottie-react-native/lottie-react-native.podspec",
          "projectPath": "/path/to/project/node_modules/lottie-react-native/src/ios/LottieReactNative.xcodeproj",
          "projectName": "LottieReactNative.xcodeproj",
          "libraryFolder": "Libraries",
          "sharedLibraries": [],
          "plist": [],
          "scriptPhases": []
        },
        "android": {
          "sourceDir": "/path/to/project/node_modules/lottie-react-native/src/android",
          "folder": "/path/to/project/node_modules/lottie-react-native",
          "packageImportPath": "import com.airbnb.android.react.lottie.LottiePackage;",
          "packageInstance": "new LottiePackage()"
        }
      },
      "assets": [],
      "hooks": {},
      "params": []
    }
  }
}
```

^ …and this is [part of] what `react-native` reports back to CocoaPods:

---

### Auto-linking

```ruby
Pod::Spec.new do |s|
  s.name         = "lottie-react-native"
  s.source_files  = "src/ios/**/*.{h,m,swift}"
  s.dependency "lottie-ios", "~> 3.1.3"
end
```

^ Given this, the helper will read the `podspec`:

---

### Auto-linking

```ruby
pod "lottie-react-native", :path => "../node_modules/lottie-react-native"
```

^ …and dynamically add the native dependency, as if it were specified like so:

---

### Auto-linking

```
$ pod install
Detected React Native module pod for lottie-react-native
Analyzing dependencies
Downloading dependencies
Installing lottie-ios (3.1.3)
Installing lottie-react-native (3.2.1)
```

^ This will in turn resolve the dependency, its transitive dependencies, and setup the Xcode projects accordingly:

---

## [fit] Demo time

---

## QA

Eloy Durán / @alloy

Bartol Karuza / @bartolkaruza

https://github.com/alloy/adding-more-native-to-your-react-native-app
