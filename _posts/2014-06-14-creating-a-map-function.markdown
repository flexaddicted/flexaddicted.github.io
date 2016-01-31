---
layout: post
title:  "Creating a map function in Objective-C"
date:   2014-06-14 00:00:00 +0000
---

I'm currently working on a project that involves JavaScript and HTML5 mobile development.
As you can imagine this is very interesting but at the same time
very challenging. You always need to pay attention to performance aspects and replicating native
behaviors could become a pain.
Even if Objective-C is my first love I have to admit that JavaScript is a very powerful language.
Around this language there are tons of wonderful libraries that help developers
in their everyday activity. Two of them are called <em>Underscore.js</em> and <em>Lo-Dash</em>. They are similar
and bring a lot of utilities including functional aspects.
A functional example included in both of them is represented by the map function. This function can be used like the following.

<pre>
_.map([1, 2, 3], function(num) { return num * 3; });
</pre>

It is applied to collections (e.g. arrays) and <em>returns a new array of the results of each
callback execution</em>. In the above example, the callback is described by

<pre>
function(num) { return num * 3; }
</pre>

while the result that is coming out is <code>[3, 6, 9]</code>.

So, is it possible to create a map "function" also in Objective-C? Obviously this question
is a rhetoric one. The approach is quite simple: creating a category around <code>NSArray</code> and using blocks.
The resulting code is the following.

<pre>
//  NSArray+FAPMap.h
typedef id(^MapBlock)(id item); // id(^)(id item)

@interface NSArray (FAPMap)

- (NSArray*)fap_map:(MapBlock)mapBlock;

@end

//  NSArray+FAPMap.m
@implementation NSArray (FAPMap)

- (NSArray*)fap_map:(MapBlock)mapBlock {

    NSMutableArray* mapArray = [NSMutableArray arrayWithCapacity:[self count]];
    for (id item in self) {
        id mappedItem = mapBlock(item);
        if(mappedItem) {
            [mapArray addObject:mappedItem];
        }
    }
    return [NSArray arrayWithArray:mapArray];
}

@end
</pre>

As it is possible to notice the <code>fap_map:</code> method has a three chars prefix in order to avoid collision with other methods.
It can be used in the following manner.

<pre>
// how to use NSArray+FAPMap category
NSArray* toMapArray = @[@1, @2, @3];
NSArray* mappedArray = [toMapArray fap_map:^id(id item) {

	NSNumber* itemAsNumber = (NSNumber*)item;
	int resultValue = [itemAsNumber integerValue] * 3;
	return @(resultValue);
}];
</pre>

Additional references: <a href="http://en.wikipedia.org/wiki/Functional_programming" target="_blank">Functional Programming</a>,
<a href="http://underscorejs.org" target="_blank">Underscore.js</a>,
<a href="http://lodash.com" target="_blank">Lo-Dash</a>
