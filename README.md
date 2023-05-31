# codebuild-perl-builder

## Intro

Most Perl modules are written entirely in Perl, but some also use XS (written in C). These
require a C compiler, as they are compiled when the Perl modules are installed.

The process of compiling the C libraries can take time, and depending on the complexity
of the library can take a signficant amount of memory. Memory your target server may not
have.

This repo contains a CodeBuild project that can be used to pre-compile Perl modules
against a particular version of Perl, and CPU architecture, so that they can
installed during build in a much shorter period of time, and with a smaller memory
footprint.

### Notes

- An Ubuntu image is used to ensure compatability with my target server
- AWS don't provide a managed arm64 image of Ubuntu, so a default image is used
- A CodeBuild project is required for each Perl module to be pre-compiled
- The tarbul is output to S3 in a folder containing the version number of Perl and the
  CPU architecture the module was built on; as these have to be identical on the target
  server, for example `5.30.0-aarch64-linux-gnu-thread-multi`.
- The only way I could find to force an ENV var is set in the `env` section, and then confirm
  that it has been overidden by the CodeBuild config (let me know if there's a better way)

## Example CodeBuild Project - Cloud Formation

See file `add-perl-builder-codebuild-project.cform`, which has just 3 mandatory parameters:

- BuildProjectName:   CodeBuild project name
- S3OutputBucketName: Bucket to store the output
- PerlModuleName:     Perl module to pre-compile

And if you want to spin up the stack using the CLI (just upload the template to S3 and update the `TEMPLATE_LOCATION` var, and update the `OUTPUT_BUCKET` to be the output location of the process)

```bash
STACK_NAME=maxmind
BUILD_PROJECT=perler-${STACK_NAME}-arm-5-30
PERL_MODULE=MaxMind::DB::Reader
OUTPUT_BUCKET=perl-build-artefacts
TEMPLATE_LOCATION=https://.....-eu-west-2.s3.eu-west-2.amazonaws.com/add-perl-builder-codebuild-project.cform

aws cloudformation create-stack --stack-name perler-${STACK_NAME} --template-url ${TEMPLATE_LOCATION} --capabilities CAPABILITY_IAM --parameters ParameterKey=S3OutputBucketName,ParameterValue=${OUTPUT_BUCKET} ParameterKey=DockerImage,ParameterValue=docker.io/ubuntu:20.04 ParameterKey=BuildEnvType,ParameterValue=ARM_CONTAINER ParameterKey=BuildComputeType,ParameterValue=BUILD_GENERAL1_SMALL ParameterKey=BuildProjectName,ParameterValue=${BUILD_PROJECT} ParameterKey=PerlModuleName,ParameterValue=${PERL_MODULE}
```

## Example CodeBuild Project - Manual

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

## Build process

- Pull and start base docker image, such as `docker.io/ubuntu.20.04` (Perl is pre-installed with Ubuntu)
- Install Perl package `cpanminus` - required for Perl package management
- Install GCC compiler and other dependencies
- Install the `Carmel` Perl module

- Create `cpanfile` with named module

- Install & compile named module and dependencies using `Carmel`
- Package compiled components

- Store in S3

## Deploy process

- Install Perl package `cpanminus`
- Install `Carmel` Perl module
- Download pre-compiled file from build stage
- Create `cpanfile`
  - `echo "requires '${PERL_MODULE_NAME}';" >> cpanfile`
- Install package using `Carmel install`

### Example

```sh
# Download files from s3
# aws s3 cp s3:\\bucket\path\file.tar.gz
# untar ...

apt-get update
apt-get upgrade -y
apt-get install gcc cpanminus -y

cpanm install Carmel --notest

echo "requires '${PERL_MODULE_NAME}';" >> cpanfile

carmel install

```
