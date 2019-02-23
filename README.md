# bKash Payment Gateway API Integration
Hello there. Welcome to unofficial guide about integrating bKash payment gateway in your application. If you already have had enough with the documentation and API references provided officially by bKash, then this documentation is just for you. Below, I will try to explain the integration steps in detail pointing all the difficulties I've faced and how I came across them. 

## Official Links 
Before digging in to the main part, lets just share the current official API documentation links available for all. These links are subjected to change without any prior notice. I will try my best to keep them updated. 
* [bKash API Reference]( https://developer.bka.sh/v1.2.0-beta/reference)
* [bKash Payment Demo](https://merchantdemo.sandbox.bka.sh/frontend/checkout)

## Preparation 

First of all, we will implement everything from the demo to our webpage. In other words, we will create a demo payment page exactly as the demo provided by the bKash and if that works as expected, we will integrate sandbox APIs with that. And, further dive into the real integration. 

To integrate bKash payment gateway, you will need jQuery and a script provided by bKash. At first, add jQuery to your webpage. Then add the bKash script. The bKash script link is not given in the demo. So, you will have to find that on your own. I will explain how to do that. Go to the demo implementation link. Open developer tools. Reload the page. You will see a request from the demo page to something like `getCheckoutScript` in the network tab. See the attached image below -

![](https://github.com/rahulhaque/bKash-payment-gateway-web-demo/blob/master/screenshots/1.PNG)

Copy the response link and replace XXX with the version you see in the third request. In this case `1.2.0-beta`. Copy paste the link you've just made in the browser and hit enter. If you see any gibberish code (encrypted JavaScript) then voila! you found the script link.

After adding the two script check in your page console by typing `bKash` and hit enter. If you don't see any error, the script is successfully loaded.

![](https://github.com/rahulhaque/bKash-payment-gateway-web-demo/blob/master/screenshots/2.PNG)

Now add a button for bKash payment exactly as the demo. Code is given below -

```html
<!-- Initially disabled -->
<button id="bKash_button" disabled="disabled">Pay With bKash</button>
```

Before diving right into the coding, we will need two more APIs that will be needed to complete the demo implementation. These are the two APIs that will be replaced with sandbox APIs and finally with the actual production ready APIs. We will call these two APIs as `createCheckoutUrl` and `executeCheckoutUrl`.

To get these two links, we will again look over to the developer tools network tab and find something like `getCheckoutMerchantBackendURL` in the request list. Again replace XXX with the version you see in the third request. See image below -

![](https://github.com/rahulhaque/bKash-payment-gateway-web-demo/blob/master/screenshots/3.PNG)

## Demo Implementation

Coding time! Just copy paste the below code in your webpage's script tag.

```javascript
let paymentID;

let createCheckoutUrl = 'https://merchantserver.sandbox.bka.sh/api/checkout/v1.2.0-beta/payment/create';
let executeCheckoutUrl = 'https://merchantserver.sandbox.bka.sh/api/checkout/v1.2.0-beta/payment/execute';

$(document).ready(function () {
    initBkash();
});

function initBkash() {
    bKash.init({
      paymentMode: 'checkout', // Performs a single checkout.
      paymentRequest: {"amount": '85.50', "intent": 'sale'},

      createRequest: function (request) {
        $.ajax({
          url: createCheckoutUrl,
          type: 'POST',
          contentType: 'application/json',
          data: JSON.stringify(request),
          success: function (data) {
              
            if (data && data.paymentID != null) {
              paymentID = data.paymentID;
              bKash.create().onSuccess(data);
            } 
            else {
              bKash.create().onError(); // Run clean up code
              alert(data.errorMessage + " Tag should be 2 digit, Length should be 2 digit, Value should be number of character mention in Length, ex. MI041234 , supported tags are MI, MW, RF");
            }

          },
          error: function () {
            bKash.create().onError(); // Run clean up code
            alert(data.errorMessage);
          }
        });
      },
      executeRequestOnAuthorization: function () {
        $.ajax({
          url: executeCheckoutUrl,
          type: 'POST',
          contentType: 'application/json',
          data: JSON.stringify({"paymentID": paymentID}),
          success: function (data) {

            if (data && data.paymentID != null) {
              // On success, perform your desired action
              alert('[SUCCESS] data : ' + JSON.stringify(data));
              window.location.href = "/success_page.html";

            } else {
              alert('[ERROR] data : ' + JSON.stringify(data));
              bKash.execute().onError();//run clean up code
            }

          },
          error: function () {
            alert('An alert has occurred during execute');
            bKash.execute().onError(); // Run clean up code
          }
        });
      },
      onClose: function () {
        alert('User has clicked the close button');
      }
    });

    $('#bKash_button').removeAttr('disabled');

}
```

Reload the page. If everything is in right order, you should see bKash payment button gets enabled. Clicking the button will show bKash popup asking for customer number just like the demo page with your given payment info. Complete the process and we will move towards implementation of the demo with sandbox APIs.

## Sandbox Implementation

For sandbox implementation, we will replace two APIs for creating and executing payment. There are some other factors to be noted. These are important as they will come in handy as we go further so I will note them down -

- All of the sandbox APIs have CORS disabled for any origins. That means you cannot access them from your browser. In order to access them, you will have to disable your browser's CORS checking completely. If you are running Windows and have Chrome installed, you press `win + r` key and type `chrome.exe --user-data-dir="C://Chrome dev session" --disable-web-security` to start Chrome in CORS disabled mode. 
- Sandbox APIs require access token for accessing and receiving response. So a grant token API call is must.
- If you haven't notice already, in the demo payment gateway, the `merchantInvoiceNumber` is auto generated by demo backend. But, in actual payment process, you will have to generate unique invoice number for each payment.

To generate access token and perform the necessary steps, you will need the following credentials provided directly by bKash -

- Sandbox username
- Sandbox password
- App key and
- App secret

Now lets have a look at the sandbox APIs that we will be needing. For that, go to the API reference link provided at the beginning of this article. As we are going to perform single checkout from our site, we will look for `Grant Token` API in the `CHECKOUT API` section. See image for clarification -

![](https://github.com/rahulhaque/bKash-payment-gateway-web-demo/blob/master/screenshots/4.PNG)

You will find create payment and execute payment sandbox APIs in the `Payment` section of the reference. See image for clarification.

![](https://github.com/rahulhaque/bKash-payment-gateway-web-demo/blob/master/screenshots/5.PNG)

After successfully getting above credentials, we need to change the code for these to be implemented. Lets add and replace these credentials in the webpage where necessary. We will also add a function to fetch the access token from grant token API. The final code will look something like this. I will comment out the changes made in the new code. See below -

```javascript
let paymentID;

let username = "Your username"; // New line
let password = "Your password"; // New line
let app_key = "Your app key"; // New line
let app_secret = "Your app secret"; // New line

let grantTokenUrl = 'https://checkout.sandbox.bka.sh/v1.2.0-beta/checkout/token/grant'; // New line
let createCheckoutUrl = 'https://checkout.sandbox.bka.sh/v1.2.0-beta/checkout/payment/create'; // Replaced API
let executeCheckoutUrl = 'https://checkout.sandbox.bka.sh/v1.2.0-beta/checkout/payment/execute'; // Replaced API

$(document).ready(function () {
    getAuthToken(); // Replaced function
});

// New function
function getAuthToken() {
    let body = {
      "app_key": app_key,
      "app_secret": app_secret
    };

    $.ajax({
      url: grantTokenUrl,
      headers: {
        "username": username,
        "password": password,
        "Content-Type": "application/json"
      },
      type: 'POST',
      data: JSON.stringify(body),
      success: function (result) {
          
        let headers = {
          "Content-Type": "application/json",
          "Authorization": result.id_token, // Contains access token
          "X-APP-Key": app_key
        };

        let request = {
            "amount": "85.50",
            "intent": "sale",
            "currency": "BDT", // New line
            "merchantInvoiceNumber": "123456" // New line
        };

        initBkash(headers, request);
      },
      error: function (error) {
        console.log(error);
      }
    });
}

function initBkash(headers, request) {
    bKash.init({
      paymentMode: 'checkout',
      paymentRequest: request, // Updated line

      createRequest: function (request) {
        $.ajax({
          url: createCheckoutUrl,
          headers: headers, // New line
          type: 'POST',
          contentType: 'application/json',
          data: JSON.stringify(request),
          success: function (data) {
              
            if (data && data.paymentID != null) {
              paymentID = data.paymentID;
              bKash.create().onSuccess(data);
            } 
            else {
              bKash.create().onError(); // Run clean up code
              alert(data.errorMessage + " Tag should be 2 digit, Length should be 2 digit, Value should be number of character mention in Length, ex. MI041234 , supported tags are MI, MW, RF");
            }

          },
          error: function () {
            bKash.create().onError(); // Run clean up code
            alert(data.errorMessage);
          }
        });
      },
      executeRequestOnAuthorization: function () {
        $.ajax({
          url: executeCheckoutUrl + '/' + paymentID, // Updated line
          headers: headers, // New line
          type: 'POST',
          contentType: 'application/json',
          success: function (data) {

            if (data && data.paymentID != null) {
              // On success, perform your desired action
              alert('[SUCCESS] data : ' + JSON.stringify(data));
              window.location.href = "/success_page.html";

            } else {
              alert('[ERROR] data : ' + JSON.stringify(data));
              bKash.execute().onError();//run clean up code
            }

          },
          error: function () {
            alert('An alert has occurred during execute');
            bKash.execute().onError(); // Run clean up code
          }
        });
      },
      onClose: function () {
        alert('User has clicked the close button');
      }
    });

    $('#bKash_button').removeAttr('disabled');

}
```

If you've done everything accordingly, you should see bKash payment popup displayed as usual with the new sandbox APIs integrated.

## Production Implementation

Last but not the least, the above setup and demo implementation is done to make the payment process as clear as possible from your point of view. We have some sensitive information in our webpage that we in no way want to share with others. We must hide them from any type of web inspection. That is why, it is recommended to call the authentication (grant token), create payment and execute payment APIs from the backend and receive the response that will be passed to the bKash script for the popup from the frontend of your application. This way, your credentials remain safe and sound increasing overall security of the app.

Lets discuss when you should make requests to your own backend server and how these requests get integrated with the bKash payment process.

1. When your frontend payment page loads, make a request to your backend for payment information. For example, pass user id or bill id through which your backend will know which payment you are talking about.
2. From your backend, generate access token by making a request to `Grant Token` API that requires `username, password, app_key` and `app_secret`. Then make a request to `Create Payment` API that includes payment related information such as - `amount, intent, currency` and `merchantInvoiceNumber` along with required header credentials. On successful payment creation, return the response to the frontend.
3. Receive the response from backend and pass that to bKash script's `bKash.create().onSuccess()` method. This will create the bKash secured popup and handle processing of the customer's phone, OTP (One Time Password) and verification pin. On successful verification, execute the payment by making a request to `Execute Payment` API. If successful, redirect the users to any success page or perform your desired action. Thus the end of integration of bKash payment gateway.

If you've come this far, then Congratulations! You've successfully integrated bKash PGW API in your application.

## Made this Far?

Phew! That was a lot of writing, isn't it? I tried my best to describe what I understood from the processes as much as possible. Spare a ★ if this helped. Take care.