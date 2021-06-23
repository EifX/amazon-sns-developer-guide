# Creating a platform application<a name="mobile-push-send-register"></a>

 For Amazon SNS to send notification messages to mobile endpoints, whether it is direct or with subscriptions to a topic, you first need to create a platform application\. After the app is registered with AWS, the next step is to create an endpoint for the app and mobile device\. The endpoint is then used by Amazon SNS for sending notification messages to the app and device\. 

**To create a platform application**

1. Sign in to the [Amazon SNS console](https://console.aws.amazon.com/sns/home)\.

1. From the navigation pane on the left, choose **Mobile** and then choose **Push notifications**\.

1. From the **Platform applications** section, choose **Create platform application**\.

   For the list of Regions where you can create mobile applications, see [Supported Regions for mobile applications](sns-mobile-push-supported-regions.md)\.

1. In the **Application name** box, enter a name to represent your app\.

   App names must be made up of only uppercase and lowercase ASCII letters, numbers, underscores, hyphens, and periods, and must be between 1 and 256 characters long\.

1. In the **Push notification platform** box, choose the platform that the app is registered with and then enter the appropriate credentials\. 
**Note**  
 If you are using one of the APNs platforms, you can select **Choose file** to upload the \.p12 file \(exported from Keychain Access\) to Amazon SNS\.

1. Choose **Create platform application**\. 

   This registers the app with Amazon SNS, which creates a platform application object for the selected platform and then returns a corresponding PlatformApplicationArn\.