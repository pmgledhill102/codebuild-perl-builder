# codebuild-perl-builder

## Intro

Most Perl modules are written in Perl, but some use XS (written in C) so require
a C compiler, and are compiled when they are installed.

The process of compiling the C libraries can take time, and depending on the complexity
of the library can take a signficant amount of memory.

This repo contains a CodeBuild stage that can be used to pre-compile Perl modules
against a particular version of Perl, and CPU architecture, so that they can
installed during build in a much shorter period of time, and with a smaller memory
footprint.

### Notes

- I wanted to use an Ubuntu image to ensure I got the right version of Perl
- I can't use an AWS managed CodeBuild image, as AWS don't provide an arm64 image of Ubuntu
- You need a CodeBuild stage for each top-level Perl module you want to pre-build
- The tarbul is output to S3 in a folder containing the version number of Perl and the
  CPU architecture the module was built on; as these have to be identical on the target
  server, for example `5.30.0-aarch64-linux-gnu-thread-multi`.
- The only way I could find to force an ENV var is set in the `env` section, and then confirm
  that it has been overidden by the CodeBuild config (let me know if there's a better way)

### Example CodeBuild Project

- Create a Build project in CodeBuild
- Project name: `Sample Perl Builder`
- Source:
  - `GitHub`
  - `Public repository`
  - URL to this repo
- Custom image:
  - `Other registry`
  - `docker.io/ubuntu:20.04`
- Environment:
  - `Custom image`
  - `ARM`
  - `Other registry`
  - `docker.io/ubuntu:20:04`
- `Additional configuration`
  - Environment Variables:
    - Name: `PERL_MODULE_NAME`
    - Value: `MaxMind::DB::Reader`
    - Type: `Plaintext`
- Buildspec
  - `Use a buildspec file`
- Artifact ` - Primary
  - Type: `Amazon S3`
  - Bucket name: `use-your-bucket-name`
  - Name: `perl-builder`

### Build process

- Pull and start base docker image, such as `docker.io/ubuntu.20.04` (Perl is pre-installed with Ubuntu)
- Install Perl package `cpanminus` - required for Perl package management
- Install GCC compiler and other dependencies
- Install the `Carmel` Perl module

- Create `cpanfile` with named module

- Install & compile named module and dependencies using `Carmel`
- Package compiled components

- Store in S3

### Deploy process

- Install Perl package `cpanminus`
- Install `Carmel` Perl module
- Download pre-compiled file from build stage
- Create `cpanfile`
  - `echo "requires '${PERL_MODULE_NAME}';" >> cpanfile`
- Install package using `Carmel install`
