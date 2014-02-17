## Sbt S3 resolver

This is an sbt plugin, which helps to resolve dependencies from and publish to Amazon S3 buckets (private or public).

## Usage

### Plugin sbt dependency

In `<your_project>/project/plugins.sbt`:

```scala
resolvers += "Era7 maven releases" at "http://releases.era7.com.s3.amazonaws.com"

addSbtPlugin("ohnosequences" % "sbt-s3-resolver" % "0.8.0")
```

#### Setting credentials

By default, credentials (the access key and secret key of your AWS account (or that of an IAM user)) are expected to be in the `~/.sbt/.s3credentials` file in the following format:

```
accessKey = 322wasa923...
secretKey = 2342xasd8fDfaa9C...
```

If you want to store your credentials somewhere else, you can set it in your `build.sbt`:

```scala
s3credentialsFile := Some("/funny/absolute/path/to/credentials.properties")
```

> don't forget to **add your credentials file to `.gitignore`**, so that you won't publish this file anywhere

As soon as you set `s3credentialsFile`, the `s3credentials` key contains the parsed credentials from that file.


### Using S3 resolver

You can construct S3 resolver using the constructor:

```scala
case class S3Resolver(
  name: String,
  url: String,
  patterns: Patterns = Resolver.defaultPatterns,
  overwrite: Boolean = false,
  region: Region = Region.EU_Ireland
)
```

Default are maven-style patterns (just as in sbt), but you can change it (setting `patterns = Resolver.ivyStylePatterns`). the parameter `overwrite = false` means that once you publish an artefact, you cannot publish it again overwriting the existing.


#### Publishing

Normal practice is to use different (snapshots and releases) repositories depending on the version. For example, here is such publishing resolver with ivy-style patterns:

```scala
publishMavenStyle := false

publishTo <<= (isSnapshot, s3credentials) { 
                (snapshot,   credentials) => 
  val prefix = if (snapshot) "snapshots" else "releases"
  // if credentials are None, publishTo is also None
  credentials map S3Resolver(
    "My "+prefix+" S3 bucket",
    "s3://"+prefix+".cool.bucket.com",
    Resolver.ivyStylePatterns
  ).toSbtResolver
}
```

You can also switch repository for public and private artifacts — you just set the url of your bucket depending on something. Note that `.toSbtResolver` in the end, this is the method which takes credentials and returns an instance of normal sbt `Resolver` type.


#### Resolving

You can add a sequence of s3 resolvers, and `flatten` it in the end, as results are `Option`s:

```scala
resolvers <++= s3credentials { cs => Seq(
    S3Resolver("Releases resolver", "s3://releases.bucket.com"),
    S3Resolver("Snapshots resolver", "s3://snapshots.bucket.com")
  ) map {r => cs map r.toSbtResolver} flatten
}
```

In this code we just convert every resolver to the function which takes credentials and do `cs map` on each to pass those credentials if they are there (it an `Option`).


#### Note about maven artifacts

This plugin can publish maven or ivy artifacts, but it can resolve only ivy-style artifacts. If your maven artifacts are public, you can resolve them using usual sbt resolvers just transforming your `s3://my.bucket.com` to

```scala
"My S3 bucket" at "http://my.bucket.com.s3.amazonaws.com"
```

i.e. without using this plugin.
