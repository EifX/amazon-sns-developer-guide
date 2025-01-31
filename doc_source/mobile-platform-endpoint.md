# Creating a platform endpoint<a name="mobile-platform-endpoint"></a>

When an app and mobile device register with a push notification service, the push notification service returns a device token\. Amazon SNS uses the device token to create a mobile endpoint, to which it can send direct push notification messages\. For more information, see [Prerequisites for Amazon SNS user notifications](sns-prerequisites-for-mobile-push-notifications.md) and [User notification process overview](sns-mobile-application-as-subscriber.md#sns-user-notifications-process-overview)\.

This section describes the recommended approach for creating a platform endpoint\.

**Topics**
+ [Create a platform endpoint](#mobile-platform-endpoint-create)
+ [Pseudo code](#mobile-platform-endpoint-pseudo-code)
+ [AWS SDK example](#mobile-platform-endpoint-sdk-examples)
+ [Troubleshooting](#mobile-platform-endpoint-problems)

## Create a platform endpoint<a name="mobile-platform-endpoint-create"></a>

To push notifications to an app with Amazon SNS, that app's device token must first be registered with Amazon SNS by calling the create platform endpoint action\. This action takes the Amazon Resource Name \(ARN\) of the platform application and the device token as parameters and returns the ARN of the created platform endpoint\.

The create platform endpoint action does the following:
+ If the platform endpoint already exists, then do not create it again\. Return to the caller the ARN of the existing platform endpoint\.
+ If the platform endpoint with the same device token but different settings already exists, then do not create it again\. Throw an exception to the caller\.
+ If the platform endpoint does not exist, then create it\. Return to the caller the ARN of the newly\-created platform endpoint\.

You should not call the create platform endpoint action immediately every time an app starts, because this approach does not always provide a working endpoint\. This can happen, for example, when an app is uninstalled and reinstalled on the same device and the endpoint for it already exists but is disabled\. A successful registration process should accomplish the following:

1. Ensure a platform endpoint exists for this app\-device combination\.

1. Ensure the device token in the platform endpoint is the latest valid device token\.

1. Ensure the platform endpoint is enabled and ready to use\.

## Pseudo code<a name="mobile-platform-endpoint-pseudo-code"></a>

The following pseudo code describes a recommended practice for creating a working, current, enabled platform endpoint in a wide variety of starting conditions\. This approach works whether this is a first time the app is being registered or not, whether the platform endpoint for this app already exists, and whether the platform endpoint is enabled, has the correct device token, and so on\. It is safe to call it multiple times in a row, as it will not create duplicate platform endpoints or change an existing platform endpoint if it is already up to date and enabled\.

```
retrieve the latest device token from the mobile operating system
if (the platform endpoint ARN is not stored)
  # this is a first-time registration
  call create platform endpoint
  store the returned platform endpoint ARN
endif

call get endpoint attributes on the platform endpoint ARN 

if (while getting the attributes a not-found exception is thrown)
  # the platform endpoint was deleted 
  call create platform endpoint with the latest device token
  store the returned platform endpoint ARN
else 
  if (the device token in the endpoint does not match the latest one) or 
      (get endpoint attributes shows the endpoint as disabled)
    call set endpoint attributes to set the latest device token and then enable the platform endpoint
  endif
endif
```

This approach can be used any time the app wants to register or re\-register itself\. It can also be used when notifying Amazon SNS of a device token change\. In this case, you can just call the action with the latest device token value\. Some points to note about this approach are:
+ There are two cases where it may call the create platform endpoint action\. It may be called at the very beginning, where the app does not know its own platform endpoint ARN, as happens during a first\-time registration\. It is also called if the initial get endpoint attributes action call fails with a not\-found exception, as would happen if the application knows its endpoint ARN but it was deleted\.
+  The get endpoint attributes action is called to verify the platform endpoint's state even if the platform endpoint was just created\. This happens when the platform endpoint already exists but is disabled\. In this case, the create platform endpoint action succeeds but does not enable the platform endpoint, so you must double\-check the state of the platform endpoint before returning success\.

## AWS SDK example<a name="mobile-platform-endpoint-sdk-examples"></a>

The following code shows how to implement the previous pseudo code using the Amazon SNS clients that are provided by the AWS SDKs\.

To use an AWS SDK, you must configure it with your credentials\. For more information, see [The shared config and credentials files](https://docs.aws.amazon.com/sdkref/latest/guide/creds-config-files.html) in the *AWS SDKs and Tools Reference Guide*\.

------
#### [ Java ]

**SDK for Java 2\.x**  
 There's more on GitHub\. Find the complete example and learn how to set up and run in the [AWS Code Examples Repository](https://github.com/awsdocs/aws-doc-sdk-examples/tree/main/javav2/example_code/sns#readme)\. 
  

```
public class RegistrationExample {

    public static void main(String[] args) {

        final String usage = "\n" +
            "Usage: " +
            "    <token>\n\n" +
            "Where:\n" +
            "   token - The name of the FIFO topic. \n\n" +
            "   platformApplicationArn - The ARN value of platform application. You can get this value from the AWS Management Console. \n\n";

        if (args.length != 2) {
            System.out.println(usage);
            System.exit(1);
        }

        String token = args[0];
        String platformApplicationArn = args[1];
        SnsClient snsClient = SnsClient.builder()
            .region(Region.US_EAST_1)
            .credentialsProvider(ProfileCredentialsProvider.create())
            .build();

        createEndpoint(snsClient, token, platformApplicationArn);
    }

    public static void createEndpoint(SnsClient snsClient, String token, String platformApplicationArn){

        System.out.println("Creating platform endpoint with token " + token);

        try {
            CreatePlatformEndpointRequest endpointRequest = CreatePlatformEndpointRequest.builder()
                .token(token)
                .platformApplicationArn(platformApplicationArn)
                .build();

            CreatePlatformEndpointResponse response = snsClient.createPlatformEndpoint(endpointRequest);
            System.out.println("The ARN of the endpoint is " + response.endpointArn());
        } catch ( SnsException e) {
            System.err.println(e.awsErrorDetails().errorMessage());
            System.exit(1);
        }
    }
}
```

------

 For more information, see [Mobile push API actions](mobile-push-api.md)\.

## Troubleshooting<a name="mobile-platform-endpoint-problems"></a>

### Repeatedly calling create platform endpoint with an outdated device token<a name="mobile-platform-endpoint-problems-outdated"></a>

Especially for FCM endpoints, you may think it is best to store the first device token the application is issued and then call the create platform endpoint with that device token every time on application startup\. This may seem correct since it frees the app from having to manage the state of the device token and Amazon SNS will automatically update the device token to its latest value\. However, this solution has a number of serious issues:
+ Amazon SNS relies on feedback from FCM to update expired device tokens to new device tokens\. FCM retains information about old device tokens for some time, but not indefinitely\. Once FCM forgets about the connection between the old device token and the new device token, Amazon SNS will no longer be able to update the device token stored in the platform endpoint to its correct value; it will just disable the platform endpoint instead\.
+ The platform application will contain multiple platform endpoints corresponding to the same device token\.
+ Amazon SNS imposes a quota on the number of platform endpoints that can be created starting with the same device token\. Eventually, the creation of new endpoints will fail with an invalid parameter exception and the following error message: "This endpoint is already registered with a different token\."

### Re\-enabling a platform endpoint associated with an invalid device token<a name="mobile-platform-endpoint-problems-invalid"></a>

When a mobile platform \(such as APNs or FCM\) informs Amazon SNS that the device token used in the publish request was invalid, Amazon SNS disables the platform endpoint associated with that device token\. Amazon SNS will then reject subsequent publishes to that device token\. While you may think it is best to simply re\-enable the platform endpoint and keep publishing, in most situations doing this will not work: the messages that are published do not get delivered and the platform endpoint becomes disabled again soon afterward\.

This is because the device token associated with the platform endpoint is genuinely invalid\. Deliveries to it cannot succeed because it no longer corresponds to any installed app\. The next time it is published to, the mobile platform will again inform Amazon SNS that the device token is invalid, and Amazon SNS will again disable the platform endpoint\.

To re\-enable a disabled platform endpoint, it needs to be associated with a valid device token \(with a set endpoint attributes action call\) and then enabled\. Only then will deliveries to that platform endpoint become successful\. The only time re\-enabling a platform endpoint without updating its device token will work is when a device token associated with that endpoint used to be invalid but then became valid again\. This can happen, for example, when an app was uninstalled and then re\-installed on the same mobile device and receives the same device token\. The approach presented above does this, making sure to only re\-enable a platform endpoint after verifying that the device token associated with it is the most current one available\.