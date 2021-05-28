---
tags: swift library
op_link: https://stackoverflow.com/q/67698422/5133585
op_profile_link: https://stackoverflow.com/users/10116367/kevvv
op_name: "Kevvv"
title: "Casting a Mysterious __SwiftValue"
---

### Premise

OP receives a response which looks like:

    ["0": 1]

When they try to get the value `1` from it, they fail:

{% highlight swift %}
let value = data["0"] as! Int
// Could not cast value of type '__SwiftValue' (0x7fff873c5bd8) to 'NSNumber' (0x7fff86d8c858).
{% endhighlight %}

They also noted that `data` is not a valid JSON object according to `JSONSerialization`.

### Is This Even Possible?

I was very confused at first. How is `["0": 1]` _not_ a valid JSON object? It clearly is! I thought maybe `data` isn't actually a dictionary, and `["0": 1]` is just the custom description of that type, or something the OP made up, based on what they think they should get.

So I asked the OP what type `data` is and where it comes from, and they said it's `[String: Any]`. Immediately I got more confused, but then I saw that they edited the question to include where they got this from. Apparently, it's from a `Promise<[String: Any]>`, and that is in turn from a library called web3swift. Very helpfully, they also provided a link to the source code of the method that they are calling.

Alright! Guess I'll do some digging then.

### Digging Through The Code

OP's link pointed me to [`callPromise`](https://github.com/skywinder/web3swift/blob/5484e81580219ea491d48e94f6aef6f18d8ec58f/Sources/web3swift/Web3/Web3%2BReadingTransaction.swift#L32), which is a method that returns a `Promise<[String: Any]>`:

{% highlight swift %}
public func callPromise(transactionOptions: TransactionOptions? = nil) -> Promise<[String: Any]>
{% endhighlight %}

I don't know much about promises. I only know it's somewhat similar to futures. I read through the code and thought that this is some Ethereum API libary (?) Anyway, I saw something along the lines of:

{% highlight swift %}
let returnPromise = Promise<[String:Any]> { seal in
    ...
    seal.fulfill(["result": resultHex as Any])
    ...
    seal.fulfill(decodedData)
    ...
}
{% endhighlight %}

I guessed that `fulfill` probably indicates the completion of the promise, or something like that. The first `fulfill` call fulfills the promise with a dictionary with only a "result" key, but OP's dictionary has a "0" key, so that's probably irrelevant. I looked at where `decodedData` comes from, and it's returnd from `self.contract.decodeReturnData(self.method, data: data)`.

I scrolled up to see that it is an `EthereumContract` object. Oh no, I thought, where on earth do I find the declaration of that? I went to the enclosing folder (web3swift/Sources/web3swift/Web3/), and saw "Web3+Contract.swift". I thought this might have it, but nope, it was a bunch of extensions on `web3`.

I looked one folder up (web3swift/Sources/web3swift/) and found a "Contracts" folder. Maybe it's in there! I looked inside and BAM, "EthereumContract.swift". I immediately found [`decodeReturnData`](https://github.com/skywinder/web3swift/blob/5484e81580219ea491d48e94f6aef6f18d8ec58f/Sources/web3swift/Contract/EthereumContract.swift#L256) by doing a text search. The only case where it can return non-nil is when `methods[method]` is a `.function`, in which case it calls `decodeReturnData` on _that_. Again, I scrolled up and saw `methods[method]` is an `ABI.Element`.

I found `ABI.Element` in web3swift/Sources/web3swift/EthereumABI/ABIElements.swift. I gotta say, this library is quite organised! In [`ABI.Element.decodeReturnData`](https://github.com/skywinder/web3swift/blob/5484e81580219ea491d48e94f6aef6f18d8ec58f/Sources/web3swift/EthereumABI/ABIElements.swift#L169), there are again two cases:

{% highlight swift %}
let name = "0"
let value = function.outputs[0].type.emptyValue
var returnArray = [String:Any]()
returnArray[name] = value
...
return returnArray
...
guard function.outputs.count*32 <= data.count else {return nil}
var returnArray = [String:Any]()
var i = 0;
guard let values = ABIDecoder.decode(types: function.outputs, data: data) else {return nil}
for output in function.outputs {
    let name = "\(i)"
    returnArray[name] = values[i]
    ...
    i = i + 1
}
return returnArray
{% endhighlight %}

The first case sets the value to some `emptyValue`, but OP's dictionary has a `1`, which doesn't seem like an "empty value", so I guessed it's the second case that's executed, which means I need to look into `ABIDecoder`. Luckily, I just saw a "ABIDecoding.swift" while I was looking for `ABI.Element`, so I found it in no time.

In [`ABIDecoder`](https://github.com/skywinder/web3swift/blob/5484e81580219ea491d48e94f6aef6f18d8ec58f/Sources/web3swift/EthereumABI/ABIDecoding.swift), I found this giant switch statement:

{% highlight swift %}
switch type {
case .uint(let bits):
    ...
    let v = BigUInt(dataSlice) % mod
    return (v as AnyObject, type.memoryUsage)
case .int(let bits):
    ...
    let v = BigInt.fromTwosComplement(data: dataSlice) % mod
    return (v as AnyObject, type.memoryUsage)
case .address:
    ...
case .bool:
    ...
case .bytes(let length):
    ...
case .string:
    ...
case .dynamicBytes:
    ...
case .array(type: let subType, length: let length):
    ...
case .tuple(types: let subTypes):
    ...
case .function:
    ...
}
{% endhighlight %}

### Aha!

The only possible cases for `1` are `uint` and `int`. I don't think a `1` would be decoded in any of the other cases, so that `1` that I saw at the beginning is actually a `BigUInt` or a `BigInt`.

I looked up what `BigInt` is, and it's apparently [another libray](https://github.com/attaswift/BigInt).

Ha! No wonder it isn't valid JSON - that `1` is a third party type!