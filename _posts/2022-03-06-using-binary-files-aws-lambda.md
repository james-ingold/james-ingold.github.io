---
title: Using Binary Files in AWS Lambda
description: Execute binary files in AWS Lambda
author: James Ingold
published: true
---

I struggled for a few hours trying to run a binary file in an AWS Lambda function. I kept getting "cannot execute binary file". I tried all sorts of fixes such as giving the file execute permissions and moving the file into the writeable tmp directory in Lambda to reapply the right permissions. The main problem turned out to be using a binary for macOS instead of the Linux version. However, I did learn some things that might help other developers save some time when running a binary file in a Lambda function.

What is a binary file?
A file that consists of compiled and executable code.

The Lambda runtime environment contains many pre-compiled binaries that can be used but sometimes you can find yourself needing to execute a binary not provide in the runtime. For example, using the aws-iam-authenticator to access a kubernetes cluster or a pdf binary to generate pdfs.

Once we have our binary file, we'll upload a zip file containing the package to be used as an Lambda Layer.

If you need to compile a binary from source, it is recommended to use an EC2 instance or the Cloud9 AWS editor. The other alternative is to use a docker container.

The format I use for my binary was a folder called package with a bin folder underneath it. package => bin with the binary placed in the bin folder.

```bash
chmod 755 /package/bin
```

You'll probably want to make sure the binary file has executable permissions.

```bash
chmod +x /package/bin/binary
```

Now zip the package

```bash
zip -r9 package.zip *
```

If you're using an EC2 or Linux server, you can pull the zip file down using scp

```bash
scp -i /key username@server:/file .
```

Upload your zip file as an AWS Lambda Layer.

A Lambda layer is a .zip file archive that can contain libraries, a custom runtime, data, or configuration files. They are used to reduce deployment package size and to promote code sharing and separation of responsibilities so that you can iterate faster on writing business logic.

In your Lambda, use the new layer and update your PATH environment variable to include the executable binary file.
\$PATH:/opt/package/bin

Now you should be able to execute the binary file in your Lambda function!
