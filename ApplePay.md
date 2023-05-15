
# Apple Pay Development book

### 1. Pre condition
   
- Set up access in [Apple Developer](https://developer.apple.com/account/resources/identifiers/list)
  refer to this [Set Up Documetion](https://developer.apple.com/documentation/passkit/apple_pay/setting_up_apple_pay) 
- Determine which processor which supoort apple pay will be used (such as Freedom pay)

  * Apple pay - Just save card info(encypted token) into device and store in the apple server
  * Payment Gateway - Handle card and send to processor(TBC)
  * Processor - Use this token to communicated with issuer bank gateway to take deduction

- Test in [Sandbox](https://developer.apple.com/apple-pay/sandbox-testing/)
```
  Note: appid need to be relatated with merchantId,and enable the capability of apple pay
   
```            
### 2. Development
- Refer to [Passkit Documentation](https://developer.apple.com/documentation/passkit/apple_pay)


```
import Foundation
import PassKit

class ApplePay: UIViewController{
  
  private var canSupportPayment: Bool = false;
  /*
   The list of available payment methods that Apple Pay supports.
  */
  private var paymentNetworks: [PKPaymentNetwork] = PKPaymentRequest.availableNetworks();
  private var rootViewController: UIViewController = UIApplication.shared.keyWindow!.rootViewController!;
  private var completion: ((PKPaymentAuthorizationResult) -> Void)?

  /*
     A bitmap field of the payment-processing protocols and card types that we support.
  */
  private var merchantCapabilities: PKMerchantCapability = [.capability3DS, .capabilityCredit];


  func initApplePay(supportedNetworks: [PKPaymentNetwork],
                    resolve: RCTPromiseResolveBlock,
                    reject: RCTPromiseRejectBlock) -> Void {


    /*
      Update paymentNetworks based on the supportedNetworks.
    */
    if supportedNetworks.count > 0 {
      self.paymentNetworks = supportedNetworks;
    }
    
    /*
      true if the device supports making payments; otherwise, false.
    */
    guard PKPaymentAuthorizationViewController.canMakePayments() else{
      resolve("APPLE_PAY_NOT_SUPPORTED");
      return;
    }
    
    
    /*
      Can make a payment by Apple pay in this devices.
    */
    self.canSupportPayment = true;

    /*
      true if the user can make payments through any of the specified networks; otherwise, false.
    */
    guard PKPaymentAuthorizationViewController.canMakePayments(usingNetworks: self.paymentNetworks, capabilities: self.merchantCapabilities) else{
      resolve("APPLE_PAY_NO_CARD");
      return;
    }
    
    resolve("APPLE_PAY_CAN_PAYMENT");}
  


  

func invokeApplePay(orderInfo:NSDictionary) -> Void{

    /*
      Check if support Apple pay, else do nothing.
    */
    guard self.canSupportPayment else{
      return;
    }
    let details = orderInfo["details"] as! NSArray
    let method = orderInfo["method"] as! NSDictionary;
    var paymentItems: [PKPaymentSummaryItem] = [];

    /*
      Make the paymentItems
    */
    for item in details {
      let item = item as! NSDictionary;
      let paymentItem = PKPaymentSummaryItem.init(label: item["label"] as! String, amount: NSDecimalNumber(value: item["value"] as! Double));
      paymentItems.append(paymentItem)
    }

    /*
      Make a payment request.
    */
    let request:PKPaymentRequest = PKPaymentRequest();
    request.currencyCode = method["currencyCode"] as! String
    request.countryCode = method["countryCode"] as! String
    request.merchantIdentifier = method["merchantIdentifier"] as! String
    request.merchantCapabilities = self.merchantCapabilities;
    request.supportedNetworks =  self.paymentNetworks;
    request.paymentSummaryItems = paymentItems;
    
    /*
      Present Apple Pay payment sheet.
    */
    if let controller = PKPaymentAuthorizationViewController(paymentRequest: request) {
      controller.delegate = self;
      DispatchQueue.main.async {
        self.rootViewController.present(controller, animated: true)
      }
    }
  }
  
  /*
    Based on server call procossor api status to update the payment sheet status.
  */
    private func updatePaymentStatus(message: String) -> Void {
      if message == "Done" {
        self.completion?(PKPaymentAuthorizationResult(status: .success, errors: []))
      }else{
        self.completion?(PKPaymentAuthorizationResult(status: .failure, errors: []))
      }
   }
}

extension ApplePay:PKPaymentAuthorizationViewControllerDelegate{
  /*
    The user authorized the payment request, and asks for a result.
  */
  func paymentAuthorizationViewControllerDidFinish(_ controller: PKPaymentAuthorizationViewController) {
    controller.dismiss(animated: true,completion: nil);
    RNEventEmitter.emitter.sendEvent(withName: "sendToken", body: "didFinish");
  }
  /*
    The user authorized the payment request, and asks for a result.
  */
  func paymentAuthorizationViewController(_ controller: PKPaymentAuthorizationViewController, didAuthorizePayment payment: PKPayment, handler completion: @escaping (PKPaymentAuthorizationResult) -> Void){
      self.completion = completion;
      let token = String(decoding: payment.token.paymentData, as: UTF8.self);
      RNEventEmitter.emitter.sendEvent(withName: "sendToken", body: token);
  }
}

```
```
import Foundation

open class RNEventEmitter : RCTEventEmitter{
  
  public static var emitter: RCTEventEmitter!;
  
  override init() {
    super.init()
    RNEventEmitter.emitter = self
  }
  
  @objc open override func supportedEvents() -> [String] {
      return ["sendToken"]
  }
};
```
### 3. Payment process
- order history
- split bill
- refund
