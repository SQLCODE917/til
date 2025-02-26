## Validating Yum

### Happy Path:

```bash
yum clean all
yum info your-name
```

Should show something like this:

```
Name         : your-name
Version      : 202502252143.23.prod
Release      : 1.50.0
Architecture : noarch
Size         : 8.4 k
Source       : your-name
Repository   : company-noarch
Summary      : your-name
License      :
Description  :
```

### But what if it doesn't?

Query Yum packages, extracts specific fields, sorts by install time, and filters for the p2-config-server package:

```bash
repoquery --all --qf "%{name}-%{version}-%{release} %{installtime} %{buildtime}" | sort -k2,2n | grep p2-config-server
```

- `sort -k2,2n` sorts by 2nd column, numerically, so look for the latest uploaded package at the bottom of the output

Look at all the other tags you can query for using `repoquery`:

```bash
 repoquery --querytags
```

What if the tag values aren't defined?
- You'd think Yum parses the filename, but no, it parses the RPM headers for this info
- Check your `gradle.build` where the tags are set and passed around lifecycle methods:

```
project.ext.release = project.findProperty("release") ?: "1.50.1"
def timestamp = new SimpleDateFormat("yyyyMMddHHmm.ss").format(new Date())
def rpmPrefix = project.findProperty("rpmPrefix")
project.ext.version = rpmPrefix ? "${timestamp}.${rpmPrefix}" : timestamp
...
task createRpm(type: Rpm,dependsOn: installDist) {
    packageName = "$rootProject.name"
    version = project.ext.version
    release = project.ext.release
}

tasks.named("buildRpm") {
    version = project.ext.version
    release = project.ext.release

    def rpmName = "${rootProject.name}-${release}-${version}.noarch.rpm"

    archiveFileName.set(rpmName)

    doLast {
        def rpmFile = archiveFile.get().asFile
        println "✅ RPM built successfully: ${rpmFile.absolutePath}"
    }
}
```

Where does `buildRpm` come from?
From [this Plugin](https://github.com/nebula-plugins/gradle-ospackage-plugin/wiki/RPM-Plugin)
