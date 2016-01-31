---
layout: post
title:  "Thoughts on creating a self-retained network lib (under ARC)"
date:   2014-05-05 00:00:00 +0000
---

In my daily work I'm now involved in JavaScript mobile development but I also want to stay updated with Objective-C.
During my train travels I read a lot but sometimes, due to the uproar or to my tiredness, I miss some important concepts.
Currently I'm reading Effective Objective-C 2.0 by Matt Galloway. A book I really suggest to everyone.
It is challenging and contains a lot of concepts.

A section of the above mentioned book talks about avoiding retain cycles introduced by block referencing the object owning them.
In particular, the author introduces a subtle retain cycle that it is not immediate to grasp. The code looks like the following.

<pre>
- (void)downloadData {
    NSURL *url = [NSURL URLWithString:@"someurlhere"];
    NetworkFetcher *networkFetcher = [[NetworkFetcher alloc] initWithURL:url];
    [networkFetcher startWithCompletionHandler:^(NSData *data){
        NSLog(@"Request URL %@ finished", networkFetcher.url);
        _fetchedData = data;
    }];
    // Here with ARC a release call will be inserted for the networkFetcher
}
</pre>

If you think in terms of object graph, the retain cycle is clear. The <code>networkFetcher</code> instance retains
the block through a <code>completionHandler</code> copyied property (I'm skipping internal details here),
while the block retains the <code>networkFetcher</code> since it uses it in <code>NSLog</code>.
Breaking the retain cycle it means setting the <code>completionHandler</code> property to nil when the request has been fulfilled.
The block will release the object it has captured, i.e. the <code>networkFetcher</code>,
when it is deallocated - which happens when the <code>completionHandler</code> property is
set to nil. So, within the <code>NetworkFetcher</code> class you need to just set as follows.

<pre>
// in NetworkFetcher
- (void)requestCompleted {
    if(self.completionHandler) {
        // invoke the block with data
        self.completionHandler(self.downloadedData);
    }

    // break the retain cycle
    self.completionHandler = nil;
}
</pre>

So, what about creating a self-retained network lib? As stated by Matt Galloway the code snippet
contained in <code>downloadData</code> method is an approach that network libraries use to keep themselves alive.
But here my question. What if I remove the line responsible for the retain cycle (i.e. the <code>NSLog</code> one)?
Under ARC, a <code>objc_release</code> call will be inserted at the end of <code>downloadData</code>
method. Hence, the request will not be performed. Since the author does not provide further details I've tried to think on it.
How does this approach work? I ended up with the following code.

<pre>
// NetworkRequest.h
typedef void(^NetworkRequestCompletionHandler)(NSData *data);

@interface NetworkRequest : NSObject

- (id)initWithURL:(NSURL*)url;
- (void)performWithCompletionHandler:(NetworkRequestCompletionHandler)completionHandler;

@end

// NetworkRequest.m
#import "NetworkRequest.h"

@interface NetworkRequest ()

@property (nonatomic, copy) NetworkRequestCompletionHandler completionHandler;
@property (nonatomic, strong) NetworkRequest *selfRef;
@property (nonatomic, strong) NSData *downloadedData;

@end

@implementation NetworkRequest

- (void)dealloc {
    NSLog(@"Hey!! NetworkRequest deallocated");
}

- (id)initWithURL:(NSURL*)url {
    if(self = [super init]) {
        _url = url;
    }
    return self;
}

- (void)performWithCompletionHandler:(NetworkRequestCompletionHandler)completionHandler {

    self.completionHandler = completionHandler;
//    self.selfRef = self;

      __weak NetworkRequest* weakSelf = self;
      dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(5 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
          [weakSelf internal_requestCompleted];
      });
}

- (void)internal_requestCompleted {

    NSLog(@"Hey!! NetworkRequest completed");

    if(self.completionHandler) {
        self.completionHandler(self.downloadedData);
    }

    self.completionHandler = nil;
//    self.selfRef = nil;
}
</pre>

Within <code>performWithCompletionHandler</code> method I set the <code>completionHandler</code> for reusing
it later and I create a sort of fake async request. Here, the using of <code>weak</code> is very important since
the dispatch_after cannot retain self. Otherwise the experiment will fail (try your own).
If now you run a <code>NetworkRequest</code> using the following code

<pre>
// how to use NetworkRequest class
NSURL *urlRequest = [NSURL URLWithString:@"someurlhere"];
NetworkRequest *request = [[NetworkRequest alloc] initWithURL:urlRequest];
[request performWithCompletionHandler:^(NSData *data) {
    NSLog(@"Hello world!!!");
}];
</pre>

you will notice that Hey!! NetworkRequest deallocated is logged as soon the app is run. This due to ARC. On the contrary,
if you remove the comments to <code>self.selfRet</code> lines the code will work as expected.

A final note is that if you don't use ARC pay attention to retain cycles...

Yep!! Goal achieved!!

Useful references: <a href="http://www.effectiveobjectivec.com" target="_blank">Effective Objective-C 2.0</a>
