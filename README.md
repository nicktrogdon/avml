# AVML (Acquire Volatile Memory for Linux)

## Summary

*A portable volatile memory acquisition tool for Linux.*

AVML is an X86_64 userland volatile memory acquisition tool written in
[Rust](https://www.rust-lang.org/), intended to be deployed as a static binary.
AVML can be used to acquire memory without knowing the target OS distribution
or kernel a priori.  No on-target compilation or fingerprinting is needed.

## Build Status

[![Build Status](https://dev.azure.com/ms/avml/_apis/build/status/microsoft.avml?branchName=main)](https://dev.azure.com/ms/avml/_build/latest?definitionId=157&branchName=main)

## Features
* Save recorded images to external locations via Azure Blob Store or HTTP PUT
* Automatic Retry (in case of network connection issues) with exponential backoff for uploading to Azure Blob Store
* Optional page level compression using [Snappy](https://google.github.io/snappy/).
* Uses [LiME](https://github.com/504ensicsLabs/LiME/) output format (when not using compression).

## Memory Sources
* /dev/crash
* /proc/kcore
* /dev/mem

If the memory source is not specified on the commandline, AVML will iterate over the memory sources to find a functional source.

## Tested Distributions
* Ubuntu: 12.04, 14.04, 16.04, 18.04, 18.10, 19.04, 19.10
* Centos: 6.5, 6.6, 6.7, 6.8, 6.9, 6.10, 7.0, 7.1, 7.2, 7.3, 7.4, 7.5, 7.6
* RHEL: 6.7, 6.8, 6.9, 7.0, 7.2, 7.3, 7.4, 7.5, 8
* Debian: 8, 9
* Oracle Linux: 6.8, 6.9, 7.3, 7.4, 7.5, 7.6

# Getting Started

## Capturing a compressed memory image

On the target host:

```
avml --compress output.lime
```

## Capturing a memory image & uploading to Azure Blob Store

On a secure host with `az cli` credentials, generate a [SAS URL](https://docs.microsoft.com/en-us/azure/storage/common/storage-sas-overview).
```
EXPIRY=$(date -d '1 day' '+%Y-%m-%dT%H:%MZ') 
SAS_URL=$(az storage blob generate-sas --account-name ACCOUNT --container CONTAINER test.lime --full-uri --permissions c --output tsv --expiry ${EXPIRY})
```

On the target host, execute avml with the generated SAS token.
```
avml --sas_url ${SAS_URL} --delete output.lime
```

## Capturing a memory image of an Azure VM using VM Extensions

On a secure host with `az cli` credentials, do the following:

1. Generate a SAS URL (see above)
2. Create `config.json` containing the following information:
```
{
    "commandToExecute": "./avml --compress --sas_url <GENERATED_SAS_URL> --delete",
    "fileUris": ["https://FULL.URL.TO.AVML.example.com/avml"]
}
```
3. Execute the [customScript](https://docs.microsoft.com/en-us/azure/virtual-machines/extensions/custom-script-linux) extension with the specified `config.json`
```
az vm extension set -g RESOURCE_GROUP --vm-name VM_NAME --publisher Microsoft.Azure.Extensions -n customScript --settings config.json
```

## To upload to AWS S3 or GCP Cloud Storage
On a secure host, generate a [S3 pre-signed URL](https://docs.aws.amazon.com/cli/latest/reference/s3/presign.html) or generate a [GCP pre-signed URL](https://cloud.google.com/storage/docs/gsutil/commands/signurl).

On the target host, execute avml with the generated pre-signed URL.
```
avml --put ${URL} --delete output.lime
```

## To decompress an AVML-compressed image
```
avml-convert ./compressed.lime ./uncompressed.lime
```

## To compress an uncompressed LiME image
```
avml-convert --format lime_compressed ./uncompressed.lime ./compressed.lime
```

# Usage

```
avml [FLAGS] [OPTIONS] <filename>

FLAGS:
        --compress    compress pages via snappy
        --delete      delete upon successful upload
    -h, --help        Prints help information
    -V, --version     Prints version information

OPTIONS:
        --sas_block_size <sas_block_size>    specify maximum block size in MiB
        --sas_url <sas_url>                  Upload via Azure Blob Store upon acquisition
        --source <source>                    specify input source [possible values: /proc/kcore, /dev/crash, /dev/mem]
        --url <url>                          Upload via HTTP PUT upon acquisition.

ARGS:
    <filename>    name of the file to write to on local system
```

# Building on Ubuntu

    # Install MUSL
    sudo apt-get install musl-dev musl-tools musl

    # Install Rust via rustup
    curl https://sh.rustup.rs -sSf | sh -s -- -y

    # Add the MUSL target for Rust
    rustup target add x86_64-unknown-linux-musl

    # Build
    cargo build --release --target x86_64-unknown-linux-musl

    # Build without upload functionality
    cargo build --release --target x86_64-unknown-linux-musl --no-default-features

# Testing on Azure

The testing scripts will create, use, and cleanup a number of resource groups, virtual machines, and a storage account.

1. Install [az cli](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)
2. Login to your Azure subscription using: `az login`
3. Build avml (see above)
4. ./test/run.sh

# Contributing

This project welcomes contributions and suggestions. Most contributions require you to
agree to a Contributor License Agreement (CLA) declaring that you have the right to,
and actually do, grant us the rights to use your contribution. For details, visit
https://cla.microsoft.com.

When you submit a pull request, a CLA-bot will automatically determine whether you need
to provide a CLA and decorate the PR appropriately (e.g., label, comment). Simply follow the
instructions provided by the bot. You will only need to do this once across all repositories using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/)
or contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.

# Reporting Security Issues

Security issues and bugs should be reported privately, via email, to the Microsoft Security
Response Center (MSRC) at [secure@microsoft.com](mailto:secure@microsoft.com). You should
receive a response within 24 hours. If for some reason you do not, please follow up via
email to ensure we received your original message. Further information, including the
[MSRC PGP](https://technet.microsoft.com/en-us/security/dn606155) key, can be found in
the [Security TechCenter](https://technet.microsoft.com/en-us/security/default).
