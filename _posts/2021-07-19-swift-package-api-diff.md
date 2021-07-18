---
layout: post
title:  "Swift Package Auto-Versioning"
date:   2021-07-18 15:01:35 +0300
image:  '/images/swift-package-api-diff/icon.png'
tags:   iOS Swift
---

Quite often, iOS Software Engineers need to encapsulate some logic to Swift packages and share it between different apps. To control the version of the code, Swift packages follow [Semantic Versioning](http://semver.org) conventions. Here is a version number format of this convention: `MAJOR.MINOR.PATCH`:

- Increase `MAJOR` version when you make incompatible API changes;
- Increase `MINOR` version when you add functionality in a backwards-compatible manner;
- Increase `PATCH` version when you make backwards compatible bug fixes.

Even though this convention is very simple to understand, there might be situations when a developer forgets about some modified parts of public code that affect the API of the framework and sets a new version of the package that doesn't represent the real changes in the framework.

What if there might be a solution that helps to automate this routine? 🤔 In this article, I will introduce you to a little tool capable of determining if changes in the Swift package are API-breaking.

## API Digester

Luckily Swift toolchain already [contains](https://github.com/apple/swift/tree/main/lib/APIDigester) such an experimental tool that can do exactly what we need. It is called **API Digester**. API Digester can work in two regimes. In the first regime, it builds and dumps the package module into a JSON file. In another, it compares two dumps and outputs the difference between them. Let's try it out!

First of all, you need to download and install Xcode 13 (beta). At the moment of writing of this article, the stable version of Xcode 12 doesn't contain this tool, but it was available before though.

Create two identical Swift packages in two directories: `old_package` and `new_package`:

{% highlight bash %}
mkdir old_package; \
cd old_package; \
swift package init --name Package; \
cd ../; \
mkdir new_package; \
cd new_package; \
swift package init --name Package; \
cd ../
{% endhighlight %}

Remove the code from Package.swift file in the new package:

{% highlight bash %}
rm new_package/Sources/Package/Package.swift; \
touch new_package/Sources/Package/Package.swift
{% endhighlight %}

Compile the packages with the Swift compiler shipped with Xcode 13:

{% highlight bash %}
SWIFT_EXEC=/Applications/Xcode-beta.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/swiftc \
swift build \
--package-path old_package \
--build-path old_build; \
SWIFT_EXEC=/Applications/Xcode-beta.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/swiftc \
swift build \
--package-path new_package \
--build-path new_build
{% endhighlight %}

Dump the packages' modules:

{% highlight bash %}
/Applications/Xcode-beta.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/swift-api-digester \
--dump-sdk \
-sdk /Applications/Xcode-beta.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk/ \
-module Package \
-I old_build/debug \
-o old_dump.json; \
/Applications/Xcode-beta.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/swift-api-digester \
--dump-sdk \
-sdk /Applications/Xcode-beta.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk/ \
-module Package \
-I new_build/debug \
-o new_dump.json
{% endhighlight %}

Compare the dumps:

{% highlight bash %}
/Applications/Xcode-beta.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/swift-api-digester \
-diagnose-sdk \
-sdk /Applications/Xcode-beta.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk/ \
--input-paths old_dump.json \
-input-paths new_dump.json
{% endhighlight %}

You will get such output:

{% highlight bash %}
/* Generic Signature Changes */

/* RawRepresentable Changes */

/* Removed Decls */
Struct Package has been removed

/* Moved Decls */

/* Renamed Decls */

/* Type Changes */

/* Decl Attribute changes */

/* Fixed-layout Type Changes */

/* Protocol Conformance Change */

/* Protocol Requirement Change */

/* Class Inheritance Change */

/* Others */
{% endhighlight %}

You see that the tool has detected that we deleted a public declaration of the `Package` struct in the new package. 

## Swift Package API Diff

In order to simplify the work with the API digester, I created a small wrapper around it called [swift-package-api-diff](https://github.com/eugenebokhan/swift-package-api-diff). The wrapper also has two commands:

- `api-changes-type` (default): get majority of api changes: breaking or non-breaking;
- `api-changes-description`: get api changes description.

Let's download and install it:

{% highlight bash %}
git clone https://github.com/eugenebokhan/swift-package-api-diff; \
cd swift-package-api-diff; \
sudo make install; \
cd ../
{% endhighlight %}

To check if the changes are breaking you can call the following command:

{% highlight bash %}
swift-package-api-diff -o old_package -n new_package -m Package
{% endhighlight %}

and the result will be `breaking`.

If you want to know what exactly has changed, you can call:

{% highlight bash %}
swift-package-api-diff api-changes-description -o old_package -n new_package -m Package
{% endhighlight %}

As a result, you will see:

{% highlight bash %}
/* Removed Decls */
 - Struct Package has been removed
{% endhighlight %}

The wrapper sanitizes the API Digester's output and shows only non-empty types changes. 

## Conclusion

The current Xcode 13's API digester is experimental and doesn't detect some changes in the packages. Another limitation of the tool is that it can work only with packages with macOS targets. But I think it's good to be familiar with such a tool and keep your eye on the ball. Someday, Apple will release Xcode with a fully working and stable API digester. Until then you can experiment with `swift-package-api-diff` and contribute to the Swift compiler to improve the tool. Thank you for reading 🙂.