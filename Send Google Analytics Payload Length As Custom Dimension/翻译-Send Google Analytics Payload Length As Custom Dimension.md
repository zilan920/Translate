Maybe you knew this, maybe you didn’t, but requests sent from your website (or app) to Google Analytics have a maximum size. Or, more specifically, the payload size (meaning the actual content body of the request) has a maximum.

This maximum size of the payload is 8192 bytes. This means, basically, that the entire parameter string sent to Google Analytics servers can be no longer than 8192 characters in length. The thing is, if the payload exceeds this, Google Analytics simply drops the hit. There’s no warning, no error, nothing. The hit just doesn’t get sent. If you are running the Google Analytics debugger browser extension, you can actually see a warning when the payload size is exceeded:

 ![Maximum Payload Size](https://www.simoahava.com/images/2018/04/ga-maximum-payload-size.png#ZgotmplZ)
If you see this warning, it means that the hit was aborted due to exceeding the 8192 length of the payload.

> Note that if you are using Measurement Protocol to directly send data to Google Analytics, this size limitation applies to the POST request body. If you are sending the data with a GET request, the maximum size of the entire /collect URL is 8000 bytes.

Anyway, in this article I’ll show you how to send the payload size (or at least a very close approximation thereof) as a Custom Dimension to Google Analytics with each hit. That way you’ll be able to check if you are approaching this limitation, and thus you can take precautions to avoid exceeding the maximum size. We’ll get things done with customTask (what else?) and Google Tag Manager.

> UPDATE: Check out Angela Grammatas’ excellent article on the very same topic. She uses a slightly different tactic, appending the data in the buildHitTask, which works just as well.

## IMPORTANT REQUEST
Before I get started, I have a request to make. Some time ago, I was contacted by someone (Googler, I think), who shared with me a similar solution. For the life of me, I can’t find this communication anywhere, because I don’t even remember what medium I was contacted over (I’ve gone through my mailbox to no avail).

This solution was definitely inspired by this person’s idea, so I want to give credit where credit is due. So, if you remember approaching me with a solution you used to work on that did something similar, please be in touch and I will update this article with my thanks for your inspirational example.

## Send the hit payload length using customTask
If you don’t know what customTask is, please check out my guide on the topic. In a nutshell, customTask lets you modify, among other things, the request sent to Google Analytics before it is sent, adding information dynamically to the payload. This information could be anything from a PII-purged payload to the Client ID.

customTask works with a Custom JavaScript variable. To send the hit payload length as a Custom Dimension, you’ll first need to create a new hit-scoped Custom Dimension in Google Analytics admin. Once you’ve created it, make note of the index assigned to the new dimension.



![Custom dimension](https://www.simoahava.com/images/2018/04/new-custom-dimension.png#ZgotmplZ)
Then, in Google Tag Manager, create a new Custom JavaScript variable. Name it something descriptive, e.g. customTask - hit payload length. This is what you should add within:
```javascript
function() {
  // Change this index to match that of the Custom Dimension you created in GA
  var customDimensionIndex = 10;
  
  return function(model) {
    
    var globalSendTaskName = '_' + model.get('trackingId') + '_sendHitTask';
    
    var originalSendHitTask = window[globalSendTaskName] = window[globalSendTaskName] || model.get('sendHitTask');
    
    model.set('sendHitTask', function(sendModel) {
      
      try {
        
        var originalHitPayload = sendModel.get('hitPayload');
        
        var hitPayload = sendModel.get('hitPayload');
        var customDimensionParameter = '&cd' + customDimensionIndex;
      
        // If hitPayload already has that Custom Dimension, note this in the console and do not overwrite the existing dimension
      
        if (hitPayload.indexOf(customDimensionParameter + '=') > -1) {
          
          console.log('Google Analytics error: tried to send hit payload length in an already assigned Custom Dimension');
          originalSendHitTask(sendModel);
        
        } else {
      
        // Otherwise add the Custom Dimension to the string
        // together with the complete length of the payload
          hitPayload += customDimensionParameter + '=';
          hitPayload += (hitPayload.length + hitPayload.length.toString().length);
        
          sendModel.set('hitPayload', hitPayload, true);
          originalSendHitTask(sendModel);
        
        }
      } catch(e) {
       
        console.error('Error sending hit payload length to Google Analytics');
        sendModel.set('hitPayload', originalHitPayload, true);
        originalSendHitTask(sendModel);
        
      }
    });
  };
}
```
To configure this customTask, you just need to update the customDimensionIndex variable with the index number of your Custom Dimension (in my example, the index is 10).

This function interrupts sendHitTask, which is the task used to actually send the request to Google Analytics. The payload sent to Google Analytics is appended with the Custom Dimension you have created, and the value of that Custom Dimension is set to the entire length of the payload. Thus, when a tag that uses this customTask fires, the request to Google Analytics will contain the length of the payload as the value of the Custom Dimension you assigned for it.

To add it to your tag(s), either use a Google Analytics Settings variable or override the settings of any tag. Scroll down to More Settings > Fields to set, and add a new field.

Field name: customTask
Value: {{customTask - hit payload length}}

 ![Fields to set](https://www.simoahava.com/images/2018/04/fields-to-set.png#ZgotmplZ)
Now any tag that has this field set will add the hit payload length as a Custom Dimension to the request.

You can verify this by opening the Network tab of your browser’s developer tools. The tag is represented by a request to /collect, and by clicking this request you can see that the Custom Dimension parameter is included with the length of the payload:

 ![Payload length in network requests](https://www.simoahava.com/images/2018/04/payload-length-custom-dimension.png#ZgotmplZ)
## Other ideas
You could actually rewrite the customTask to add some further logic if you are nearing the 8192 byte maximum size. For example, it doesn’t really help you to add the payload length as a Custom Dimension, if the payload is never sent due to exceeding the size requirement.

By adding some custom code, you could do things like:

When payload length is at or over 8192 bytes, drop unnecessary fields from the payload until the length is under the limit.

When payload length is at or over 8192 bytes, send a new hit to Google Analytics which contains some key information from the “broken” payload, just so that you’ll get an idea of the scope of the problem.

A typical reason for exceeding payload length is if you are sending Enhanced Ecommerce impression data, or just very large shopping carts. It only takes something like 50-60 products for you to be approaching the limit of 8192 bytes. Unless you can decrease the size of the product payload using regular means, you could do something like drop all non-critical fields from the product objects (e.g. brand, category, variant), and just include the id, name, price, and quantity.

Or you could drop all product data except for id in these cases, and then use Data Import to refresh the additional data.

## Summary
Well, it looks like customTask to the rescue again. It’s such a powerful feature, and I just love the fact that you can get all meta with your Google Analytics data.

Peppering the payload with details that are otherwise obfuscated is a great way to add a whole new level of debugging opportunities to your data collection.

Hit payload size limit is nasty in that you won’t be alerted if a problem surfaces. The Google Analytics UI won’t be able to inform you about this problem because the requests are never sent to GA in the first place. With this Custom Dimension, you can add this information to the payload as a Custom Dimension, and then prepare in advance for the eventuality that you might be hitting the payload maximum size sometime soon.