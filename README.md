Maven Central Release Action
============================

[![GitHub Actions Marketplace][GitHub Actions Marketplace badge]][GitHub Actions Marketplace URL]
[![GitHub Workflow Status][GitHub Workflow Status badge]][GitHub Workflow Status URL]
[![Apache License][Apache License badge]][Apache License URL]

<!-- TOC -->

- [Maven Central Release Action](#maven-central-release-action)
  - [Deploying Maven Artifacts to Maven Central in One Action](#deploying-maven-artifacts-to-maven-central-in-one-action)
  - [How to Use Maven Central Release Action](#how-to-use-maven-central-release-action)
    - [Step 1 - Push Tag](#step-1---push-tag)
    - [Step 2 - Create GPG Key](#step-2---create-gpg-key)
      - [Installing GnuPG](#installing-gnupg)
      - [Generating a Key Pair](#generating-a-key-pair)
    - [Step 3 - Distributing Public Key](#step-3---distributing-public-key)
    - [Step 4 - Configuring Maven Central Credentials](#step-4---configuring-maven-central-credentials)
    - [Step 5 - Preparing POM File](#step-5---preparing-pom-file)
      - [A Complete Example POM](#a-complete-example-pom)
    - [Step 6 - Defining Action File](#step-6---defining-action-file)
      - [Optional Parameters](#optional-parameters)
  - [Troubleshooting](#troubleshooting)
    - ["Invalid signature for file"](#invalid-signature-for-file)
    - [keyserver send/receive failed](#keyserver-sendreceive-failed)
  - [License](#license)

<!-- TOC -->

Deploying Maven Artifacts to Maven Central in One Action
--------------------------------------------------------

**Maven Central Release Action** offers a convenient "one-click" approach for publishing artifacts to
[Maven Central](https://central.sonatype.com/)

How to Use Maven Central Release Action
---------------------------------------

### Step 1 - Push Tag

Manually create the first tag **on `master` branch**. For example, `1.0.0`:

```console
git tag -a 1.0.0 -m "1.0.0"
git push origin 1.0.0
```

*We recommend all-number version such as "1.0.0" as opposed to "v1.0.0"*

> [!TIP]
> When the release is done, the action will automatically create and push a new version tag of
> `MAJOR`.`MINOR`.(`PATCH` + 1)
>
> Bumping the `MAJOR` or `MINOR` version still needs to be done manually using `git tag -a vx.x.x -m "vx.x.x"` command
> given the assumption that agile software development will change patch version most frequently and almost always

### Step 2 - Create GPG Key

One of the [requirements](https://central.sonatype.org/publish/requirements/) for publishing our artifacts to the
Central Repository, is that they have been signed with PGP. [GnuPG or GPG](http://www.gnupg.org/) is a freely available
implementation of the OpenPGP standard. GPG provides us with the capability to generate a signature, manage keys, and
verify signatures.

#### Installing GnuPG

[Download the binary of GnuPG](https://gnupg.org/download/index.html#sec-1-2) or install it with our favorite package
manager and verify it by running a gpg command with the `--version` flag

```console
$ gpg --version
gpg (GnuPG) 2.2.19
libgcrypt 1.8.5
Copyright (C) 2019 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <https://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Home: /home/mylocaluser/.gnupg
Supported algorithms:
Pubkey: RSA, ELG, DSA, ECDH, ECDSA, EDDSA
Cipher: IDEA, 3DES, CAST5, BLOWFISH, AES, AES192, AES256, TWOFISH,
        CAMELLIA128, CAMELLIA192, CAMELLIA256
Hash: SHA1, RIPEMD160, SHA256, SHA384, SHA512, SHA224
Compression: Uncompressed, ZIP, ZLIB, BZIP2
```

#### Generating a Key Pair

A key pair allows us to sign artifacts with GPG and users can subsequently validate that artifacts have been signed by
us. We can generate a key with:

```console
gpg --gen-key
```

Enter our name and email when asked for it and also, the time of validity for the key defaults to 2 years. Once they
key is expired we can extend it, provided we own the key and therefore know the passphrase.

> [!CAUTION]
> Please keep in deep mind that **the GPG key name entered cannot contain white space!**

```console
$ gpg --gen-key
gpg (GnuPG) 2.2.19; Copyright (C) 2019 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Note: Use "gpg --full-generate-key" for a full featured key generation dialog.

GnuPG needs to construct a user ID to identify your key.

Real name: my-gpg-keyname
Email address: central@example.com
You selected this USER-ID:
    "my-gpg-keyname <central@example.com>"

Change (N)ame, (E)mail, or (O)kay/(Q)uit? O
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.
gpg: key 8190C4130ABA0F98 marked as ultimately trusted
gpg: revocation certificate stored as
'/home/mylocaluser/.gnupg/openpgp-revocs.d/CA925CD6C9E8D064FF05B4728190C4130ABA0F98.rev'
public and secret key created and signed.

pub   rsa3072 2021-06-23 [SC] [expires: 2023-06-23]
      CA925CD6C9E8D064FF05B4728190C4130ABA0F98
uid                      my-gpg-keyname <central@example.com>
sub   rsa3072 2021-06-23 [E] [expires: 2023-06-23]
```

We have to provide our name and email. These identifiers are essential as they will be seen by anyone downloading a
software artifact and validating a signature. Finally, we must provide a **passphrase** to protect our secret key. It
is essential that we choose a secure passphrase and that we do not divulge it to any one. This passphrase and **gpg's
private key** are all that is needed to sign artifacts with our signature.

> [!TIP]
> - **[Create a GitHub Secret] named *GPG_PASSPHRASE* whose value is the passphrase**
> - **[Create a GitHub Secret] named *GPG_KEYNAME* whose value, in the example above, is the "*my-gpg-keyname*" shown
> above**

Please keep in deep mind that **GPG_KEYNAME cannot contain white space!**

> [!CAUTION]
> Please allow me to press it again - keep in mind that **GPG_KEYNAME cannot contain white space!**

To export the private key, list it along with any other keys installed:

```console
$ gpg --list-keys
/home/mylocaluser/.gnupg/pubring.kbx
---------------------------------
pub   rsa3072 2021-06-23 [SC] [expires: 2023-06-23]
      CA925CD6C9E8D064FF05B4728190C4130ABA0F98
uid           [ultimate] my-gpg-keyname <central@example.com>
sub   rsa3072 2021-06-23 [E] [expires: 2023-06-23]
```

The output displays the path to the public keyring file. The line starting with `pub` shows the size (rsa3072), the
**keyid** (`CA925CD6C9E8D064FF05B4728190C4130ABA0F98`), and the creation date (2023-06-23) of the public key. Some
values may vary depending on our GnuPG version, but we will definitely see the keyid or part of it (called shortID,
last 8 characters of the keyid, in this example `0ABA0F98`, which we can ask gpg to output using
`gpg --list-keys --keyid-format short`).

The next line shows the UID of the key, which is composed of a name, a comment, and an email.

Next we [export an ascii armored version of the private key](https://unix.stackexchange.com/a/482559/437808):

```console
gpg --output private.pgp --armor --export-secret-key CA925CD6C9E8D064FF05B4728190C4130ABA0F98
```

> [!TIP]
> **[Create a GitHub Secret] named *GPG_PRIVATE_KEY* whose value is the entire output the command above**

### Step 3 - Distributing Public Key

The maven-central-release-action would need the public key to verify the files, we want to distribute GPG public key to
a key server:

```console
gpg --keyserver keyserver.ubuntu.com --send-keys CA925CD6C9E8D064FF05B4728190C4130ABA0F98
```

> [!IMPORTANT]
> The GPG Keyservers supported by Central Servers are:
> `keyserver.ubuntu.com`
> `keys.openpgp.org`
> `pgp.mit.edu`

The `--keyserver` parameter identifies the target key server address. The `--send-keys` is the keyid of the key we want
to distribute. We can get our keyid by [listing the public keys](#invalid-signature-for-file).

Now the action can import our public key from the key server to CI/CD server:

```console
gpg --keyserver keyserver.ubuntu.com --recv-keys CA925CD6C9E8D064FF05B4728190C4130ABA0F98
```

### Step 4 - Configuring Maven Central Credentials

Releasing to Maven Central requires credentials by generating a user token via the [Maven Central account page].

> [!TIP]
> - **[Create a GitHub Secret] named *MAVEN_CENTRAL_USERNAME* whose value is the token username**
> - **[Create a GitHub Secret] named *MAVEN_CENTRAL_TOKEN* whose value is the token password**

### Step 5 - Preparing POM File

As part of the deployment, we are required to submit a POM file. This is the Project Object Model file used by Apache
Maven to define our project and its build. When building with other tools we have to assemble it and ensure it contains
the following information.

- **Correct Coordinates**: The project coordinates, also known as GAV, coordinates determine the location of your
  project in the repository. The values are

  - `groupId`: the top level namespace level for our project starting with the reverse domain name
  - `artifactId`: the unique *name* for our artifact
  - `version`: the version string for our artifact

  The version can be an arbitrary string and can not end in `-SNAPSHOT`, since this is the reserved string used to
  identify versions that are currently in development. We must use [semantic versioning](http://semver.org) such as
  1.0.0

  A valid example is

  ```xml
  <groupId>com.example.applications</groupId>
  <artifactId>example-application</artifactId>
  <version>1.4.7</version>
  ```

- **Project Name, Description and URL**: For some human-readable information about our project and a pointer to our
  project website for more, we need the presence of `name`, `description`, and `url`:

  ```xml
  <name>Example Application</name>
  <description>
      A application used as an example on how to set up pushing its components to the Central Repository.
  </description>
  <url>http://www.example.com/example-application</url>
  ```

- **License Information** - We need to declare the license(s) used for distributing our artifacts. E.g. if we use the
  Apache License we can use

  ```xml
  <licenses>
      <license>
          <name>The Apache Software License, Version 2.0</name>
          <url>http://www.apache.org/licenses/LICENSE-2.0.txt</url>
          <distribution>repo</distribution>
      </license>
  </licenses>
  ```

- **Developer Information** - In order to be able to associate the project it is required to add a developers section.

  ```xml
  <developers>
      <developer>
          <name>Paion Data Team</name>
          <url>https://github.com/paion-data</url>
      </developer>
  </developers>
  ```

- **SCM Information**

  ```xml
  <scm>
      <developerConnection>scm:git:ssh://git@github.com/github-username/repo-name.git</developerConnection>
      <url>https://github.com/github-username/repo-name.git</url>
      <tag>HEAD</tag>
  </scm>
  ```

  **Please modify the `github-username` and `repo-name` above accordinly**

#### A Complete Example POM

```xml
<?xml version="1.0" encoding="UTF-8"?>

<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.paiondata.athena</groupId>
    <artifactId>athena-parent-pom</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>

    <name>Athena: Parent POM</name>
    <url>https://github.com/paiondata/athena</url>
    <description>Athena: Parent POM</description>

    <developers>
        <developer>
            <name>Paion Data Team</name>
            <url>https://github.com/paion-data</url>
        </developer>
    </developers>

    <licenses>
        <license>
            <name>The Apache Software License, Version 2.0</name>
            <url>http://www.apache.org/licenses/LICENSE-2.0.txt</url>
            <distribution>repo</distribution>
        </license>
    </licenses>

    <scm>
        <developerConnection>scm:git:ssh://git@github.com/paion-data/athena.git</developerConnection>
        <url>https://github.com/paion-data/athena.git</url>
        <tag>HEAD</tag>
    </scm>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-javadoc-plugin</artifactId>
                <version>3.5.0</version>
                <configuration>
                    <doclint>none</doclint>  <!-- Turnoff all checks -->
                </configuration>
                <executions>
                    <execution>
                        <id>attach-javadocs</id>
                        <goals>
                            <goal>jar</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-source-plugin</artifactId>
                <version>3.1.0</version>
                <executions>
                    <execution>
                        <id>attach-sources</id>
                        <phase>verify</phase>
                        <goals>
                            <goal>jar-no-fork</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

    <profiles>
        <profile>
            <id>release</id>
            <build>
                <plugins>
                    <plugin>
                        <groupId>org.sonatype.central</groupId>
                        <artifactId>central-publishing-maven-plugin</artifactId>
                        <version>0.4.0</version>
                        <extensions>true</extensions>
                        <configuration>
                            <tokenAuth>true</tokenAuth>
                            <autoPublish>true</autoPublish>
                            <publishingServerId>${gpg.keyname}</publishingServerId>
                        </configuration>
                    </plugin>
                    <plugin>
                        <groupId>org.apache.maven.plugins</groupId>
                        <artifactId>maven-gpg-plugin</artifactId>
                        <version>3.1.0</version>
                        <executions>
                            <execution>
                                <id>sign-artifacts</id>
                                <phase>verify</phase>
                                <goals>
                                    <goal>sign</goal>
                                </goals>
                                <configuration>
                                    <gpgArguments>
                                        <arg>--pinentry-mode</arg>
                                        <arg>loopback</arg>
                                    </gpgArguments>
                                    <keyname>${gpg.keyname}</keyname>
                                    <passphraseServerId>${gpg.keyname}</passphraseServerId>
                                </configuration>
                            </execution>
                        </executions>
                    </plugin>
                </plugins>
            </build>
        </profile>
    </profiles>
</project>
```

> [!NOTE]
> Note that we've configured project above to use the [central-publishing-maven-plugin] and [Apache Maven GPG Plugin].
>
> In addition, projects with packaging other than pom have to supply JAR files that contain Javadoc and sources. This
> allows the consumers of our artifacts to automatic access to Javadoc and sources for browsing as well as for display
> and navigation e.g. in their IDE.

### Step 6 - Defining Action File

Under regular `.github/workflows` directory, create a `.yml` file with a preferred name with the following example
contents:

```yaml
---
name: My App CI/CD

"on":
  pull_request:
  push:
    branches:
      - master

jobs:
  release:
    name: Release to Maven Central
    if: github.ref == 'refs/heads/master'
    runs-on: ubuntu-latest
    steps:
      - name: Release
        uses: paion-data/maven-central-release-action@master
        with:
          user: QubitPi
          email: jack20220723@gmail.com
          gpg-keyname: ${{ secrets.GPG_KEYNAME }}
          gpg-private-key: ${{ secrets.GPG_PRIVATE_KEY }}
          gpg-passphrase: ${{ secrets.GPG_PASSPHRASE }}
          server-username: ${{ secrets.MAVEN_CENTRAL_USERNAME }}
          server-password: ${{ secrets.MAVEN_CENTRAL_TOKEN }}
```

> [!NOTE]
> More information can be found at [The Central Repository Documentation](https://central.sonatype.org/)

#### Optional Parameters

- `version-properties`: The names of all properties (separated by comma, not white space between them) that contains
  the artifact version. For example, we might have a POM property called

  ```yaml
  <properties>
      <myproject.version>1.0-SNAPSHOT</myproject.version>
  </properties>
  ```

  then we will need to pass `myproject.version` via this option in order to fully set the release properties. Internally
  the action utilizes [Versions Maven Plugin] to update all specified properties to the lastly tagged version

Troubleshooting
---------------

### "Invalid signature for file"

```console
- Invalid signature for file: ***-1.0.0-javadoc.jar
- Invalid signature for file: ***-1.0.0-sources.jar
- Invalid signature for file: ***-1.0.0-tests.jar
- Invalid signature for file: ***-1.0.0.jar
- Invalid signature for file: ***-1.0.0.pom
```

There could be multiple causes for this error. **Most likely one or more of the following GPG parameters are not
specified correctly**:

- **gpg-keyname**:
- **gpg-private-key**
- **gpg-passphrase**

Please note that the `gpg-keyname` in the following example is *my-gpg-keyname*:

```console
$ gpg --list-keys
/home/mylocaluser/.gnupg/pubring.kbx
---------------------------------
pub   rsa3072 2021-06-23 [SC] [expires: 2023-06-23]
      CA925CD6C9E8D064FF05B4728190C4130ABA0F98
uid           [ultimate] my-gpg-keyname <central@example.com>
sub   rsa3072 2021-06-23 [E] [expires: 2023-06-23]
```

If everything seems correct, we could try deleting and [re-generating](#step-2---create-gpg-key) a new GPG key

> [!TIP]
> There are two types of GPG keys:
>
> 1. **Public keys** This type of key ensures data encryption and is used to validate the origin of a message. Public
>    keys are meant to be shared openly as the message can be decrypted only with the corresponding private key.
> 2. **Private keys** This type of key should be kept confidential for security reasons. A single private key is paired
>    with a single public key counterpart and both are necessary for authentication and decryption.
>
> To list public keys in Linux, run:
>
> ```console
> gpg --list-keys
> ```
>
> To list private keys, use gpg with the --list-secret-keys option:
>
> ```console
> gpg --list-secret-keys
> ```
>
> To ensure successful key removal, **delete the private key first and then proceed with deleting the public key**:
>
> ```console
> gpg --delete-secret-key CA925CD6C9E8D064FF05B4728190C4130ABA0F98
> gpg --delete-key CA925CD6C9E8D064FF05B4728190C4130ABA0F98
> ```

### keyserver send/receive failed

[Try](https://unix.stackexchange.com/questions/361642/keyserver-receive-failed-on-every-keyserver-available#comment1032068_383783)

```console
sudo pkill dirmngr
```

or [send GPG key online](https://keyserver.ubuntu.com/) (assuming `keyserver.ubuntu.com` keyserver)

License
-------

The use and distribution terms for [maven-central-release-action] are covered by the [Apache License, Version 2.0].

[Apache License, Version 2.0]: http://www.apache.org/licenses/LICENSE-2.0.html
[Apache License badge]: https://img.shields.io/badge/Apache%202.0-F25910.svg?style=for-the-badge&logo=Apache&logoColor=white
[Apache License URL]: https://www.apache.org/licenses/LICENSE-2.0
[Apache Maven GPG Plugin]: https://maven.apache.org/plugins/maven-gpg-plugin/usage.html#configure-passphrase-in-settings-xml-with-a-keyname

[central-publishing-maven-plugin]: https://central.sonatype.com/artifact/org.sonatype.central/central-publishing-maven-plugin
[Create a GitHub Secret]: https://docs.github.com/en/actions/security-guides/encrypted-secrets

[GitHub Actions Marketplace badge]: https://img.shields.io/badge/View%20On%20Marketplace-2088FF?style=for-the-badge&logo=githubactions&logoColor=white
[GitHub Actions Marketplace URL]: https://github.com/marketplace/actions/maven-central-release
[GitHub Workflow Status badge]: https://img.shields.io/github/actions/workflow/status/paion-data/maven-central-release-action/ci-cd.yml?branch=master&logo=github&style=for-the-badge
[GitHub Workflow Status URL]: https://github.com/paion-data/maven-central-release-action/actions/workflows/ci-cd.yml

[maven-central-release-action]: https://github.com/QubitPi/maven-central-release-action
[Maven Central account page]: https://central.sonatype.com/account

[Versions Maven Plugin]: https://www.mojohaus.org/versions/versions-maven-plugin/update-properties-mojo.html
