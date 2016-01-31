---
layout: post
title:  "Extending NSManaged object classes"
date:   2014-05-02 00:00:00 +0000
---

In Core Data, as described in the Apple documentation, <code>insertNewObjectForEntityForName:inManagedObjectContext:</code> makes it easy for you to create instances of a given entity without worrying about the details of managed object creation. This method requires two parameters: a name for the entity you want to create and a valid context. About the context you have different ways to grab/retrieve it (maybe I will talk about in a future post). Concerning the name, having an hardcoded string all over the place it is error prone. You need to be sure the entity name has the correct syntax. In addition, this approach makes your code not easy to maintain - what happens if you want to rename a specific entity?

Fortunately there are different ways to solve the problem.

The first one is to create a category around <code>NSManagedObject</code>. For example, your category would look like this:

<pre>

// NSManagedObject+CDAdditions.h
@interface NSManagedObject (CDAdditions)

+ (NSString *)CDAdditionsEntityName;
+ (instancetype)CDAdditionsInsertNewObjectIntoContext:(NSManagedObjectContext *)context;

@end

// NSManagedObject+CDAdditions.m
@implementation NSManagedObject (CDAdditions)

+ (NSString *)CDAdditionsEntityName {    
    return NSStringFromClass(self);
}

+ (instancetype)CDAdditionsInsertNewObjectIntoContext:(NSManagedObjectContext *)context {    
    return [NSEntityDescription insertNewObjectForEntityForName:[self CDAdditionsEntityName] inManagedObjectContext:context];
}

@end

</pre>

An alternative way is to directly subclass <code>NSManagedObject</code>. For example:

<pre>

// CDBaseManagedObject.h
@interface CDBaseManagedObject : NSManagedObject

+ (NSString *)entityName;
+ (instancetype)insertNewObjectIntoContext:(NSManagedObjectContext *)context;

@end

// CDBaseManagedObject.m
@implementation CDBaseManagedObject

+ (NSString *)entityName {    
    return NSStringFromClass(self);
}

+ (instancetype)insertNewObjectIntoContext:(NSManagedObjectContext *)context {    
    return [NSEntityDescription insertNewObjectForEntityForName:[self entityName] inManagedObjectContext:context];
}

@end

</pre>

As last but not least there is also <a href="http://rentzsch.github.io/mogenerator/" target="_blank">mogenerator</a>. This amazing tool generates classes based on your data model providing this functionality and much more.

Choose what you prefer!!!

Useful references: <a href="http://vimeo.com/89370886" target="_blank">Core Data Potpourri</a>,
<a href="https://github.com/mmorey/MDMCoreData" target="_blank">MDMCoreData</a>
