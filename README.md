# Sams ESP32 AWS IoT example using Arduino SDK

![Build Status](https://github.com/jandelgado/esp32-aws-iot/workflows/run%20tests/badge.svg)

This is a fork of https://github.com/ExploreEmbedded/Hornbill-Examples
focusing on the AWS_IOT library and removing everything else. The code was
also upgraded to AWS IoT Device SDK v3.0.1.

The library was modified so that the TLS configuration (i.e. certificates and
stuff) is _no longer_ included in the library code self, but is now passed to
the `AWS_IOT` class from the client code. This makes the library easier usable.

* original repository:  https://github.com/ExploreEmbedded/Hornbill-Examples

Additionally, I added a tutorial on using the AWS cli to create everything
needed to set your thing up. 

Also I added [all-in-one thing creator
script](#all-in-one-thing-creator-script) which uses the AWS API (python/boto3)
to create a thing, certificates and attach certificates and policies. In
addition the script outputs ready-to-include C++ code to be included in your
sketch.

## Contents

<!-- vim-markdown-toc GFM -->

* [Examples](#examples)
    * [pubSubTest Example/Quickstart](#pubsubtest-examplequickstart)
        * [Build](#build)
* [AWS IoT core notes](#aws-iot-core-notes)
    * [Create thing group and thing](#create-thing-group-and-thing)
    * [Create keys and certificates](#create-keys-and-certificates)
    * [Attach policy to your thing](#attach-policy-to-your-thing)
    * [All-in-one thing creator script](#all-in-one-thing-creator-script)
    * [MQTT endpoint](#mqtt-endpoint)
* [Troubleshooting](#troubleshooting)
    * [Error -0x2780](#error--0x2780)
    * [Error -0x2700](#error--0x2700)
* [Author](#author)

<!-- vim-markdown-toc -->

## Examples

### pubSubTest Example/Quickstart

Under [examples/pubSubTest](examples/pubSubTest) the original PubSub example of
the hornbill repository is included, with the configuration externalized to a
separate file [config.h-dist](examples/pubSubTest/config.h-dist). To build the
example, copy the `config.h-dist` file to a file called `config.h` and modify
to fit your configuration:

* add your WiFi configuration
* add AWS thing private key (see below)
* add AWS thing certificate (see below)
* add [AWS MQTT endpoint address](#mqtt-endpoint)

The easisiet way to obtain the certificates and create the things is to use
the [all-in-one thing creator script](#all-in-one-thing-creator-script). Don't
forget to create a IAM policy first (see below).

#### Build

A plattformio [project](platformio.ini) and [Makefile](Makefile) is provided.

Run `make upload monitor` to build and upload the example to the ESP32 and
start the serial monitor afterwards to see what is going on.

If everything works, you should see something like this on your console:
```
Attempting to connect to SSID: MYSSID
Connected to wifi
Connected to AWS
Subscribe Successfull
Publish Message:Hello from hornbill ESP32 : 0
Received Message:Hello from hornbill ESP32 : 0
Publish Message:Hello from hornbill ESP32 : 1
Received Message:Hello from hornbill ESP32 : 1
...
```

## AWS IoT core notes

**Work in progress**

This chapter describes how to use the AWS cli to

* create a thing type and thing
* create keys and certificates for a thing
* attach a certificate to a thing
* create and attach a policy to a thing

### Create thing group and thing

We use the AWS cli to create a thing type called `ESP32` and a thing with the
name `ESP32_SENSOR`. For convenience and later reference, we store the things
name in the environment variable `THING`.

```bash
$ export THING="ESP32_SENSOR"
$ aws iot create-thing-type --thing-type-name "ESP32"
$ aws iot list-thing-types
{
    "thingTypes": [
        {
            "thingTypeName": "ESP32",
            "thingTypeProperties": {
                "thingTypeDescription": "ESP32 devices"
            },
            "thingTypeMetadata": {
                "deprecated": false,
                "creationDate": 1530358342.86
            },
            "thingTypeArn": "arn:aws:iot:eu-central-1:*****:thingtype/ESP32"
        }
    ]
}
$ aws iot create-thing --thing-name "$THING" --thing-type-name "ESP32"
$ aws iot list-things
{
    "things": [
        {
            "thingTypeName": "ESP32",
            "thingArn": "arn:aws:iot:eu-central-1:*****:thing/ESP32_SENSOR",
            "version": 1,
            "thingName": "ESP32_SENSOR",
            "attributes": {}
        }
    ]
}
```

### Create keys and certificates

```bash
$ aws iot create-keys-and-certificate --set-as-active \
                                      --certificate-pem-outfile="${THING}_cert.pem" \
                                      --private-key-outfile="${THING}_key.pem"
{
    "certificateArn": "arn:aws:iot:eu-central-1:*****:cert/7bb8fd75139186deef4c054a73d15ea9e2a5f603a29025e453057bbe70c767fe",
    "certificatePem": "-----BEGIN CERTIFICATE-----\n ... \n-----END CERTIFICATE-----\n",
    "keyPair": {
        "PublicKey": "-----BEGIN PUBLIC KEY-----\n ... \n-----END PUBLIC KEY-----\n",
        "PrivateKey": "-----BEGIN RSA PRIVATE KEY-----\n ... \n-----END RSA PRIVATE KEY-----\n"
    },
    "certificateId": "7bb8fd75139186deef4c054a73d15ea9e2a5f603a29025e453057bbe70c767fe"
}

```

On the thing we later need (see [config.h-dir](examples/pubSubTest/config.h-dist)):

* the private key stored in `${THING}_key.pem` (i.e. `ESP32_SENSOR_key.pem`)
* the certificate stored in `${THING}_cert.pem` (i.e. `ESP32_SENSOR_cert.pem`)

Note that this is the only time that the private key will be output by AWS.

Next we attach the certificate to the thing:

```bash
$ aws iot attach-thing-principal --thing-name "$THING" \
         --principal "arn:aws:iot:eu-central-1:*****:cert/7bb8fd75139186deef4c054a73d15ea9e2a5f603a29025e453057bbe70c767fe"
$ aws iot list-principal-things --principal  "arn:aws:iot:eu-central-1:*****:cert/7bb8fd75139186deef4c054a73d15ea9e2a5f603a29025e453057bbe70c767fe"
{
    "things": [
        "ESP32_SENSOR"
    ]
}
aws iot list-thing-principals --thing-name $THING
{
    "principals": [
        "arn:aws:iot:eu-central-1:*****:cert/7bb8fd75139186deef4c054a73d15ea9e2a5f603a29025e453057bbe70c767fe"
    ]
}
```

### Attach policy to your thing

It is important to attach a policy to your thing (technically: to the
certificate attached to the thing), otherwise no communication will be
possible.

First we need to create a permission named `iot-full-permissions` which, as
the name suggests, has full iot permissions:

```bash
$ cat >thing_policy_full_permissions.json<<EOT
{
   "Version" : "2012-10-17",
   "Statement" : [
      {
         "Action" : [
            "iot:Publish",
            "iot:Connect",
            "iot:Receive",
            "iot:Subscribe",
            "iot:GetThingShadow",
            "iot:DeleteThingShadow",
            "iot:UpdateThingShadow"
         ],
         "Effect" : "Allow",
         "Resource" : [
            "*"
         ]
      }
   ]
}
EOT
$ aws iot create-policy --policy-name "iot-full-permissions" \
                        --policy-document file://thing_policy_full_permissions.json
$ aws iot list-policies
{
    "policies": [
        {
            "policyName": "iot-full-permissions",
            "policyArn": "arn:aws:iot:eu-central-1:*****:policy/iot-full-permissions"
        }
    ]
}
```

(TODO least privilege permission sets in policies)

Next, we attach the policy to the certificate:

**WiP**

```bash
"arn:aws:iot:eu-central-1:*****:thing/ESP32_SENSOR"
$ aws iot attach-policy --policy-name "iot-full-permissions"  \
                        --target "7bb8fd75139186deef4c054a73d15ea9e2a5f603a29025e453057bbe70c767fe"
$ aws iot list-targets-for-policy --policy-name iot-full-permissions
...
```

### All-in-one thing creator script

Entering above aws cli commands manually is slow and error prone. See the
provided [create_thing.py](tools/create_thing/create_thing.py) Python script,
which performs all steps automatically and also produces c++ code containing
the certificate and keys ready to be included in your ESP32 sketch.

```
usage: create_thing.py [-h] [--type-name TYPE_NAME] name policy_name

positional arguments:
  name                  name of the thing to create
  policy_name           policy to attach thing to

optional arguments:
  -h, --help            show this help message and exit
  --type-name TYPE_NAME
                        thing type name
```

### MQTT endpoint

Running `aws describe-endpoint` will give you the endpoint of your MQTT service
(make sure to use the `ATS` enpoint for the data service type):

```bash
$ aws iot describe-endpoint --endpoint-type iot:Data-ATS
{
    "endpointAddress": "*****-ats.iot.eu-central-1.amazonaws.com"
}
```

The secure MQTT port is 8883. To test if your MQTT endpoint is up, you could
for example issue a netcat command like `nc -v <your-iot-endpoint> 8883`.

## Troubleshooting

### Error -0x2780

```
E (15426) aws_iot: failed!  mbedtls_x509_crt_parse returned -0x2780 while parsing device cert`
```

Check the format of your PEM key and certificates in `config.h`. 

### Error -0x2700

```
(11417) aws_iot: failed! mbedtls_ssl_handshake returned -0x2700
E (11417) aws_iot:     Unable to verify the server's certificate. 
E (11427) AWS_IOT: Error(-4) connecting to **********.iot.eu-central-1.amazonaws.com:8883,
```

Are you using the correct MQTT endpoint (hint use the `ATS` endpoint)?

## Author

Jan Delgado <jdelgado at gmx.net>, original work from https://github.com/ExploreEmbedded/Hornbill-Examples.

