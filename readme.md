# Verifying a Phone Number in iOS using Swift 

In this tutorial you learn how to verify a user's phone number using our Swift framework. We support two methods of verifying on iOS:

* Classic [SMS PIN Verification](https://www.sinch.com/products/verification/sms-verification/)
* [Phone Verification](https://www.sinch.com/products/verification/) where we place a callout to a number and the user then presses 1 to confirm that they wanted the call

We also offer [Flash Call Verification](https://www.sinch.com/products/verification/flash-call-verification/) but this is only available for Android.

At the end of this tutorial we will have a basic app that looks like this:

<center><img src="images/screen1.png" width="290px" alt="enable verification"><img src="images/screen2.png" width="290px" alt="successful verification"></center>

##Getting Started

We've made a video walkthough to get you started with this project. Incudes how to set up the Xcode project, implement SMS and Callout verification, and some debugging at the end. Click the image to watch the video.

<a href="https://www.youtube.com/watch?v=fu1b3awU8I8" target="_blank"><img src="images/video.png" alt="Verify a phone number in Swift" border="10" /></a>

###Sinch Setup
1. [Create an account](https://www.sinch.com/dashboard/#/signup)
2. Create an app and change set authorization to public (for now)
![Sinch dashboard](images/dashboard.png)
3. Take note of the application key 

Head over to Xcode to get some basic setup.

###Xcode Setup
1. Create a new single view project 
2. Add a cocoapods file and install pods
3. A physical device with SIM card

```text
platform :ios, '8.0'
use_frameworks! 
pod 'SinchVerification-Swift'
```

##Building The First Screen UI
Open up your workspace and go to the **main.storyboard** file. Then open up the assistant view to also see the **ViewController.swift**. Then follow these steps:

1. Add a textfield and add an outlet called **phoneNumber**. Set the keyboard type of the field to phone number
2. Add an SMS verification button and create an action called **smsVerification** 
3. Add a callout verification button and create an action called **calloutVerification** 
4. Add a label and call an outlet called **status**
5. Add an activity indicator and an outlet called **spinner**, and then check the box to hide when no animated
6. Embed the ViewController in a navigation controller editor

Add your constraints and then the first screen is done. The next thing we are going to do is to add some code to do a callout verification. I want to start with this because the callout verification does not require any additional screens.

##Callout Verification
The verification flow for a callout is pretty neat. Just imitate a callout verification and when the callback comes to the client, that's it. 

How does it work? Sinch will place a call to the given phone number and when the user picks up we prompt the user to press 1. If they do, it's treated as a success, but if they don't then it is treated as a fail (or if they don't pick up, etc).

Open up **ViewController.swift** and add an import for **SinchVerification**:

```swift
import SinchVerification;
```

At the top of your class, add two variables to hold the verification and one to hold the application key. 

```swift
var verification:Verification!;
var applicationKey = "your key";
```

Great! Now we want to start a callout verification once the user clicks on the callout verification button. 

```swift
@IBAction fun calloutVerification(sender: AnyObject) {
    disableUI(true);
    verification = CalloutVerification(
    	applicationKey:applicationKey, 
    	phoneNumber: phoneNumber.text!);
    verification.initiate { (success:Bool, error:NSError?) -> Void in
        self.disableUI(false);
        self.status.text = (success ? "Verified" : error?.description);
    }
}
```

As you can see that's not a lot of code to make this roll. You might have noticed that I have a **disbleUI(Bool)** call in there, and that's a small method to disable the UI while waiting for the call. This is important to do because if the user starts multiple verification requests they might get stuck in a loop and might never get verified and the phone would just keep ringing. I've implemented a timeout of 30 seconds before I consider it to be a fail and the user can try again. 

```swift
fun disableUI(disable: Bool){
    var alpha:CGFloat = 1.0; // if enabled alpha is 1
    if (disable) {
        alpha = 0.5; add alpha to get disabled look
        phoneNumber.resignFirstResponder(); 
        spinner.startAnimating(); 
        self.status.text="";
        // enable the UI after 30 seconds if no success or fail has been received in the
        //verification sdk
        let delayTime = dispatch_time(DISPATCH_TIME_NOW, Int64(30 * Double(NSEC_PER_SEC)))
        dispatch_after(delayTime, dispatch_get_main_queue(), { () -> Void in
            self.disableUI(false);
        });
    }
    else{
        self.phoneNumber.becomeFirstResponder();
        self.spinner.stopAnimating();
        
    }
    self.phoneNumber.enabled = !disable;
    self.smsButton.enabled = !disable;
    self.calloutButton.enabled = !disable;
    self.calloutButton.alpha = alpha;
    self.smsButton.alpha = alpha;
}
```

Now we can add some improved UI. Start by adding a **viewWillAppear** and set the phone number to first responder.

```swift
    override func viewWillAppear(animated: Bool) {
        phoneNumber.becomeFirstResponder();
        disableUI(false); // make sure UI is enabled,
    }
```

So nothing fancy as you can see. Run the app and try it out. Pretty sweet right?

##Adding SMS Verification

Another way of adding verification is the classic SMS PIN that I am sure you have used. The downside of SMS in my opinion is that you need to enter a code which adds some friction to the user experience. 

To accomplish an SMS verification, you will need a new view where a user can enter a code. Add a new **ViewController** to the solution and call in **EnterCodeViewController.swift**. 

Open up **Main.Storyboard** and add a view controller to the board and set the first responder to the newly created Controller:

1. Add a textfield and an outlet called **code** 
2. Add a button and an action called **verify**
3. Add label and an outlet called **status**
4. Lastly a spinner and an outlet called **spinner** 
5. Add a segue from **ViewController.swift** to **EnterCodeViewController.swift** and call it **enterPin**.

Add your constraints and make it look how you want, but it should look something like this:

<center><img src="images/screen2.png" width="290px" alt="successful verification"></center>

### Initiating an SMS verification
Initiating an SMS verification is very similar to Callout. The big difference here is when you get the success callback, it doesn't mean its verified, it just means that we have sent an SMS. What we need to do after that is to send in a code that we get from user input to verify it. In this case we do that in a separate view controller. Once we have the success, we perform the segue to show the entertain controller.   

```swift
@IBAction fund smsVerification(sender: AnyObject) {
	self.disableUI(true);
	verification = 
		SMSVerification(applicationKey:applicationKey, 
			phoneNumber: phoneNumber.text!)
	verification.initiate { (success:Bool, error:NSError?) -> Void in
	    self.disableUI(false);
	    if (success){
	        self.performSegueWithIdentifier("enterPin", sender: sender);
	    } else {
	        self.status.text = error?.description;
	    }
	}
}
```

To verify a verification, you need to keep the current verification object, so in in **prepareForSegue** we want to pass on the current verification object so we can call verify on it. 

```swift
override fun prepareForSegue(segue: UIStoryboardSegue, sender: AnyObject!) {
    if (segue.identifier == "enterPin") {
        let enterCodeVC = segue.destinationViewController as! EnterCodeViewController;
        enterCodeVC.verification = self.verification;
    }
    
}
```

Now that's out of the way, open up **EnterCodeViewController.swift** and go to the action **verify** and set up the UI for making a verification request, and call verify on your verification object. 

```swift
@IBAction fund verify(sender: AnyObject) {
	spinner.startAnimating();
	verifyButton.enabled = false;
	status.text  = "";
	pinCode.enabled = false;
	verification.verify(pinCode.text!, 
		completion: { (success:Bool, error:NSError?) -> Void in
	   		self.spinner.stopAnimating();
	    	self.verifyButton.enabled = true;
	    	self.pinCode.enabled = true;
	    if (success) {
	        self.status.text = "Verified";
	    } else {
	        self.status.text = error?.description;
	    }
	});
}
```

## Finished

You now have a verified number for your user. With this implementation you only know on the client side that the number is verified. In a real world app, you would need to tell your backend that the number is verified. You could accomplish that in two ways. Either calling that update on the success flow from the client. Or your own callbacks that we have for verification (recommended).

For more details, check out our [verification documentation](https://www.sinch.com/docs/verification/rest#verificationcallbackapi).
