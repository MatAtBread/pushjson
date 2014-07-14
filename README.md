pushjson
========

A really asynchronous JSON stringinfy good for making Nodejs co-operative when sending big JSON responses

Installation
============

	npm install pushjson

Usage
=====
The easiest way to use pushjson is to call its Readable method to create a Readable stream from an object and simply pipe to a destination. This works for almost all real use-cases.

The 'writeToStream' method takes a Writable (or Duplex) stream as it's first parameter and sends in chunks. It tries to cope with the various Stream implementations in node v10+, some of which are incomplete or broken (e.g OutgoingMessage.write does neither buffering or accepts a final "cbDone" parameter when it should). 

Finally, you can pass a function to 'writeToStream' instead of a Stream that is called when the serializer has some data to send. It optionally provides a second parameter (quite frequently in practice) that allows you to either suspend sending and continue later using the method 'continueSending', or continue sending immediately without yielding control back to the Node event loop.


	var json = require('pushjson') ;
		...
	// Create a Readable stream from an object and pipe it to a response
	json.Readable(objectToSerailize,serializer).pipe(response) ;
		...
	// Write directly to a stream and close it when we're finished
	json.writeToStream(response,objectToSerailize,serializer,function(){ 
		rsp.end(); 
	}) ;
		...
	// Intercept write requests and handle them "manually"
	json.writeToStream(function send(data,sendNext){
		if (!sendNext) {
			// This is a request to just send a bit of data to whatever
			// we're doing with it
			handleSerializedData(data) ;
		} else {
			// This is a request to send some data, and it gives us an object
			// which we can either return directly to send the next bit, or
			// defer and do later via json.continueSending(sendNext);
			if (canHandleSerializedData(data)) {
				handleSerializedData(data) ;
				// Do the next bit of the object
				return sendNext ; 
			} else {
				// Asynchronously send more data. In the best scenario,
				// our reciever will notifiy us when it's ready for more data
				// by some kind of event or callback, or we can just
				// do it on a timer or similar
				setImmediate(json.continueSending(sendNext)) ;
			}
		}
		
	},objectToSerailize,serializer,function(){ 
		rsp.end(); 
	}) ;
		...

Issues
======
pushjson will probably do bad things if you mutate the object while it is serializing in the "background". It is really only intended for the final, terminal phase of JSON object serializtion.

pushjson starts with with a "reasonable" sized buffer so that short objects will never yield. In this case, serialization times are about 20% longer than the native JSON.stringify. Typical sizes where it really helps are hundreds of KB, when the native JSON.stringify takes a few hundred milliseconds or more. Note that Node's event loop and IO model mean that overall sending times for big JSON objects are not great - pushJSON does not make it faster, it makes it more co-operative meaning you can send big JSON data to more clients at once, rather than have them processed consecutively.

pushjson, like the native JSON, supports toJSON() methods on objects being serialized, and "replacer" argument (https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_native_JSON#The_replacer_parameter). It does not support the "space" parameter for generating "pretty" JSON. The replacer parameter does not currently support the "Array of field names" implementation, only the callback function.
 




