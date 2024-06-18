*©2023 Axis Communications AB. AXIS COMMUNICATIONS, AXIS, ARTPEC and VAPIX are registered trademarks of Axis AB in various jurisdictions. All other trademarks are the property of their respective owners.*

<!-- omit from toc -->
# This tutorial has been archived, and its content moved to [Axis Developer Documentation](https://developer.axis.com/computer-vision/how-to-guides/build-hard-hat-detection-using-axis-object-analytics-and-amazon-recognition)

<!-- omit from toc -->
# Hybrid machine learning - Hard hat detection using AXIS Object Analytics and Amazon Rekognition

<!-- omit from toc -->
## Table of contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Solution setup](#solution-setup)
    - [Camera image to Amazon S3 bucket](#camera-image-to-amazon-s3-bucket)
    - [AWS IoT Core and MQTT client](#aws-iot-core-and-mqtt-client)
    - [Amazon Rekognition PPE detection (inference)](#amazon-rekognition-ppe-detection-inference)
    - [AXIS Object Analytics (camera application) and event setup](#axis-object-analytics-camera-application-and-event-setup)
    - [Strobe siren event and MQTT subscription](#strobe-siren-event-and-mqtt-subscription)
- [Test and validation](#test-and-validation)
- [Disclaimer](#disclaimer)
- [License](#license)

## Overview

Hybrid machine learning refers to the use of edge computing (on-premise) for initial data processing (in this setup the Axis camera), and then sending the more compute-intensive tasks to the cloud for deeper analysis. This approach is commonly used in scenarios where you might want to reduce the amount of data sent to the cloud, manage latency, or address privacy and security concerns.

This tutorial describes a solution that uses [AXIS Object Analytics](https://www.axis.com/products/axis-object-analytics) on the Axis camera to do the initial data processing, only sending images to [Amazon Web Services (AWS)](https://aws.amazon.com/) when people enters the scene. Pre-trained models in [Amazon Rekognition](https://aws.amazon.com/rekognition/) assesses that the people are using personal protective equipment (PPE), such as a helmet. The inference result is sent to [AXIS D4100-E Network Strobe Siren](https://www.axis.com/products/axis-d4100-e-network-strobe-siren) via MQTT, that either signals green to indicate that the helmet is on and red to indicate that the helmet is off.

![PPE detection video](assets/ppe_video.gif)

The architectural overview below shows the required components (hardware and software) and protocols to set up the solution.

> [!NOTE]
> You can replace some components to develop solutions for other use cases. For example, replace AXIS Object Analytics with another application to send images to AWS. Or replace the Axis strobe siren with an Axis door station to change the output and result of the solution.
>
> Modifying the AWS Lambda function that calls the Amazon Rekognition is also possible if the use case requires other detection types.

![Solution overview](assets/architecture-overview.png)

## Prerequisites

- Camera running [AXIS Object Analytics](https://www.axis.com/products/axis-object-analytics) - Click [here](https://www.axis.com/products/axis-object-analytics#compatible-products) to find cameras featuring the *line crossing* scenario.
- [AXIS D4100-E Network Strobe Siren](https://www.axis.com/products/axis-d4100-e-network-strobe-siren)
- Access to AWS - Amazon Rekognition isn't available in all regions. For this setup to work, select one of its [supported regions](https://docs.aws.amazon.com/general/latest/gr/rekognition.html).

## Solution setup

These are the main steps of this tutorial:

- Camera image to Amazon Simple Storage Service (Amazon S3) bucket
- AWS IoT Core, acting as a MQTT broker
- Amazon Rekognition PPE detection (inference)
- AXIS Object Analytics (camera application) and event setup
- Strobe siren event and MQTT subscription

![architecture](assets/architecture.png)\
*Detailed overview showing all cloud services.*

### Camera image to Amazon S3 bucket

The image below illustrates the upload of a camera image to an Amazon S3 bucket.

![Image upload to Amazon S3](assets/aws-image-upload.png)

[Sending images from a camera to Amazon S3](https://github.com/AxisCommunications/acap-integration-examples-aws/tree/main/images-to-aws-s3) describes how to set up the Amazon S3 and the required peripheral services to handle the authentication.

> [!NOTE]
> Follow the instructions up until the section called **Configure the camera**. Keep note on the AWS CloudFormation stack output parameters named `Recipient`, `AccessToken` and `Bucket` as you will need them later on in this tutorial.

### AWS IoT Core and MQTT client

In this section we'll set up AWS IoT Core and have the Axis strobe siren connect to it using its MQTT client.

![AWS IoT Core](assets/aws-iot-core.png)

#### Set up an AWS IoT Core thing

1. Sign in to the [AWS Management Console](https://aws.amazon.com/console/) and search for **IoT Core**.
2. Go to **Manage** > **All devices** > **Things** and click **Create things**.
3. Select **Create single thing** and click **Next**.
4. Enter a unique name and click **Next**.
5. On the **Configure device certificate** page, select **Auto-generate a new certificate** and click **Next**.
6. Create a new policy or attach an existing one to the certificate. You're redirected to a new page if you create a new policy. For this tutorial, create a new policy with three statements:
    - First statement
        - **Policy effect**: `Allow`
        - **Policy action**: `iot:Connect`
        - **Policy resource**: `*`
    - Second statement
        - **Policy effect**: `Allow`
        - **Policy action**: `iot:Subscribe`
        - **Policy resource**: `*`
    - Third statement
        - **Policy effect**: `Allow`
        - **Policy action**: `iot:Receive`
        - **Policy resource**: `*`

    >[!WARNING]
    > Not restricting the policy resource is acceptable in an exploratory phase, but applying [least-privilege permissions](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html#grant-least-privilege) is a requirement before going to production.

7. Return to the previous page to attach the applicable policy and click **Create thing**.
8. Download the **Device certificate**, **Public key file**, **Private key file** and the **Root CA certificate**.

    ![download aws certificates](assets/aws-download-certificates.png)\
    *Screenshot from AWS Management Console*

9. Click **Done**.

#### Set up the MQTT client in the Axis strobe siren

In the Axis strobe siren, install the client and CA certificates to enable a secure MQTT connection to AWS IoT Core:

1. Log in to the Axis device and go to **System** > **Security**.
2. Click **Add certificate**.
3. Select **Upload a client-server certificate using a separate private key** and click **Next**.
4. Upload the client certificate (filename ends with `certificate.pem.crt`) and the private key (filename ends with `private.pem.key`) and click **Next**.
5. Click **Install** and then **Close**.
6. Click **Add certificate** again and select **Upload a CA certificate**.
7. Upload the root CA certificate (`AmazonRootCA1.pem`).
8. Click **Next** and then **Install** and **Close**.

Next, configure the device's MQTT client:

1. In the Axis device go to **System** > **MQTT**.
2. In the **Host** field, enter the hostname for AWS IoT Core. You can find the hostname (device data endpoint) in the AWS Management Console under **IoT Core** > **Settings**.
3. In the **Protocol** drop-down menu, select **MQTT over SSL** using the default port **8883**.
4. In the **Client certificate** field, select the previously uploaded client certificate.
5. In the **CA certificate** field, select the previously uploaded CA certificate.
6. Select **Validate server certificate**.
7. Click **Save**.
8. Turn on **Connect**.

Here's an example of an **MQTT client** setup in an Axis device.

![axis device mqtt client settings](assets/axis-device-mqtt-client-settings.png)\
*©2023 Axis Communications AB. All rights reserved.*

### Amazon Rekognition PPE detection (inference)

This section explains how to set up and configure the AWS Lambda function to grab an image from the Amazon S3 bucket, input the image to Amazon Rekognition, and transfer the detection result (helmet on or off) to AWS IoT Core.

![AWS Lambda and Amazon Rekognition](assets/aws-lambda_rekognition.png)

#### Create an AWS Lambda function

1. Sign in to the [AWS Management Console](https://aws.amazon.com/console/) and search for Lambda.
2. Create a new AWS Lambda function.
    - Select **Author from scratch**.
    - Set a name for the function.
    - Select **Node.js 18.x** as runtime.
    - Select **x86_64** as architecture.
    - Click **Create function**.

#### Set up the AWS Lambda function

1. Add a trigger to the Amazon S3 bucket where you store the uploaded images.
2. Add the code below to the `index.mjs` file within your AWS Lambda function.

    ```javascript
    import { RekognitionClient, DetectProtectiveEquipmentCommand } from "@aws-sdk/client-rekognition";
    import { IoTDataPlaneClient, PublishCommand } from "@aws-sdk/client-iot-data-plane";

    const BUCKET_NAME = process.env.BUCKET_NAME;
    const IOT_CORE_DEVICE_DATA_ENDPOINT = process.env.IOT_CORE_DEVICE_DATA_ENDPOINT;

    const rekognitionClient = new RekognitionClient();
    const ioTDataPlaneClient = new IoTDataPlaneClient({
        endpoint: `https://${IOT_CORE_DEVICE_DATA_ENDPOINT}`,
    });

    const anyPersonWithoutProtectiveEquipment = async (objectKey) => {
        const input = {
            Image: {
                S3Object: {
                    Bucket: BUCKET_NAME,
                    Name: objectKey,
                },
            },
            SummarizationAttributes: {
                MinConfidence: 50,
                RequiredEquipmentTypes: ["HEAD_COVER"],
            },
        };

        const command = new DetectProtectiveEquipmentCommand(input);

        const res = await rekognitionClient.send(command);
        return res.Summary.PersonsWithoutRequiredEquipment.length > 0;
    };

    const publishMessage = async (alarm) => {
        const input = {
            topic: alarm ? "ppe/alarm/on" : "ppe/alarm/off",
            payload: JSON.stringify({ message: "PPE detection payload" }),
        };

        const command = new PublishCommand(input);

        await ioTDataPlaneClient.send(command);
    };

    export const handler = async (event) => {
        if (!event || !event.Records || event.Records.length !== 1) {
            console.log(`unexpected event structure: ${JSON.stringify(event)}`);
            return;
        }

        const objectKey = event.Records[0].s3.object.key;
        console.log(`bucket: ${BUCKET_NAME}, object key: ${objectKey}`);

        const alarm = await anyPersonWithoutProtectiveEquipment(objectKey);
        await publishMessage(alarm);
    };
    ```

3. Go to **Configuration** of the AWS Lambda function and set up two **Environment variables**. One for the bucket name and one for the AWS IoT Core endpoint.

![AWS Lambda, Environment variables](assets/aws-env-var.png)\
*Screenshot from AWS Management Console*

#### AWS Lambda function permissions

Finally, set up the correct permissions for the AWS Lambda function to access the Amazon S3 bucket, Amazon Rekognition and AWS IoT Core.

1. Go to **Configuration** > **Permissions** and click the **Role name** in **Execution role**.
2. In the **IAM** (Identity and Access Management) role, select **Create inline policy** from the **Add permissions** dropdown.
3. Specify the following permissions and create the policy:
    - **Service**: `S3`
        - `GetObject`
        - `GetObjectVersion`
    - **Service**: `Rekognition`
        - `DetectProtectiveEquipment`
    - **Service**: `IoT`
        - `Publish`
4. With the policy created and attached to the role, you're done configuring the AWS Lambda permissions.

### AXIS Object Analytics (camera application) and event setup

#### Line Crossing in AXIS Object Analytics

1. In the Axis camera, go to **Apps** and make sure that the application named `AXIS Object Analytics` is running.
2. Click **Open** to configure the application.
3. Set up a line crossing scenario where you want to capture an image and send it to Amazon S3.

![AXIS Object Analytic - Line crossing](assets/axis-line-crossing.png)\
*Example of a line crossing scenario in AXIS Object Analytics*

#### Set up Axis camera event

Now it is time to set up an HTTPS recipient to the Amazon API Gateway and an event that triggers an image upload.

1. In the Axis camera go to **System** > **Events**.
2. On the **Recipients** tab click **Add recipient**, pointing to the Amazon API Gateway.
    - **Name**: `AWS S3`
    - **Type**: `HTTPS`
    - **URL**: The AWS CloudFormation stack output parameter named `Recipient`, from back when we created the Amazon S3 bucket.
3. On the **Rules** tab click **Add a rule** with the following settings:
    - **Name**: `AXIS Object Analytics: Line crossing`
    - **Condition**: The Object Analytics scenario created in AXIS Object Analytics
    - **Action**: `Send images through HTTPS`
        - **Recipient**: `AWS S3`
        - **Maximum images**: `1`
        - **Custom CGI parameters**: The AWS CloudFormation stack output parameter named `AccessToken` points to a secret in AWS Secrets Manager. Follow the link, retrieve the secret value and format it according to `accessToken=<secret value>`. E.g. if the secret value is `FooBar` then the value you should enter here is `accessToken=FooBar`. Please note that copying the secret from AWS Secrets Manager sometimes adds a space in the beginning of the secret. Please make sure to remove this space before saving the rule.

The camera will now send an image every time a person crosses the line that you created in AXIS Object Analytics.

### Strobe siren event and MQTT subscription

This section explains how to set up the MQTT subscription in the Axis strobe siren and tie the MQTT topic to different events that control the strobe siren's light or sound.

#### Set up MQTT subscription

1. In the strobe siren under **System** > **MQTT** go to **MQTT subscriptions** and add a new subscription, matching one of the message topics sent by the Amazon Rekognition AWS Lambda function.
    - **Subscription filter**: `ppe/alarm/on`
    - **Use default topic prefix**: `Unchecked`
2. Add a second subscription, matching the other message topics sent by the Amazon Rekognition AWS Lambda function.
    - **Subscription filter**: `ppe/alarm/off`
    - **Use default topic prefix**: `Unchecked`

#### Set up a strobe siren profile

1. In the strobe siren add a new profile, give it a name (example `green-light`), and configure the desired signaling.
2. Create a second profile for some other color or sound (example `red-light`).

#### Set up an event

1. In the strobe siren, go to **System** > **Events** and add a rule with the following settings:
    - **Name**: `Alarm on`
    - **Condition**:
        - **Type**: `Stateless`
        - **Subscription filter**: `ppe/alarm/on`
        - **Use device topic prefix**: `Unchecked`
    - **Action**
        - **Type**: `Run light and siren profile`
        - **Profile**: `red-light`
        - **Action**: `Start`
2. Add a second rule with the following settings:
    - **Name**: `Alarm off`
    - **Condition**:
        - **Type**: `Stateless`
        - **Subscription filter**: `ppe/alarm/off`
        - **Use device topic prefix**: `Unchecked`
    - **Action**
        - **Type**: `Run light and siren profile`
        - **Profile**: `green-light`
        - **Action**: `Start`

## Test and validation

To test the solution, trigger the line crossing in AXIS Object Analytics. The Axis strobe siren should light up red or green, depending on whether you're wearing a helmet.

To check the MQTT messages sent to AWS IoT Core, log in to the AWS Management Console and navigate to the MQTT test client in AWS IoT Core. You can see all messages published to the broker if you subscribe using the wildcard `#`.

![AWS MQTT test client](assets/aws-mqtt-test-client.png)\
*Screenshot from AWS Management Console*

## Disclaimer

<!-- textlint-disable -->

This document and its content are provided courtesy of Axis and all rights to the document shall remain vested in Axis Communications AB. AXIS COMMUNICATIONS, AXIS, ARTPEC and VAPIX are registered trademarks of Axis AB in various jurisdictions. Amazon Web Services, AWS and the Powered by AWS logo, Amazon Simple Storage Service (Amazon S3), Amazon Rekognition and AWS Lambda are trademarks of Amazon.com, Inc. or its affiliates. All other trademarks are the property of their respective owners, and we are not affiliated with, endorsed or sponsored by them or their affiliates.

As described in this document, you may be able to connect to, access and use third party products, web sites, example code, software or services (“Third Party Services”). You acknowledge that any such connection and access to such Third Party Services are made available to you for convenience only. Axis does not endorse any Third Party Services, nor does Axis make any representations or provide any warranties whatsoever with respect to any Third Party Services, and Axis specifically disclaims any liability or obligations with regard to Third Party Services. The Third Party Services are provided to you in accordance with their respective terms and conditions, and you alone are responsible for ensuring that you (a) procure appropriate rights to access and use any such Third Party Services and (b) comply with the terms and conditions applicable to its use.

PLEASE BE ADVISED THAT THIS DOCUMENT IS PROVIDED “AS IS” WITHOUT WARRANTY OF ANY KIND, AND IS NOT INTENDED TO, AND SHALL NOT, CREATE ANY LEGAL OBLIGATION FOR AXIS COMMUNICATIONS AB AND/OR ANY OF ITS AFFILIATES. THE ENTIRE RISK AS TO THE USE, RESULTS AND PERFORMANCE OF THIS DOCUMENT AND ANY THIRD PARTY SERVICES REFERENCED HEREIN IS ASSUMED BY THE USER OF THE DOCUMENT AND AXIS DISCLAIMS AND EXCLUDES, TO THE MAXIMUM EXTENT PERMITTED BY LAW, ALL WARRANTIES, WHETHER STATUTORY, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO ANY IMPLIED WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE, TITLE AND NON-INFRINGEMENT AND PRODUCT LIABILITY.

<!-- textlint-enable -->

## License

Example Code means the examples provided by Axis in this document within the grey text boxes.

Example Code ©2023 Axis Communications AB is licensed under the [Apache License, Version 2.0 (the “License”)](./LICENSE). You may not use the Example Code except in compliance with the License.

You may obtain a copy of the License at <https://www.apache.org/licenses/LICENSE-2.0>. Example Code distributed under the License is distributed on an “AS IS” BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.
