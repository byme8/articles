Always when I was reading about the Roslyn and analyzers that come with it, I always cought myself on thought: "Wait a second! Looks like I can use Rosyn analyzer to create nuget package that will be able to analyzer my source code and code generate some parts of application right inside the IDE." Brief investigation in Google gave nothing, so I decided to dig in to investigate how it can be done. I was suprised when figure out that it is possible and it is going to be relatively easy to do.

So iff you want to see how to make 'small reflection' and ship it as nuget package this article is for you.

<cut/>

# Introduction

I guess it's  better to explane what do I mean under term 'small reflection'. I want to make method ``` Dictionary<string, Type> GetBakedType() ``` that you can call on any type. It is going to return property names and thier types. Because it have to work with all types the easiest way is to create separate extension method for each type. The manual implemantation will look like that:

``` cs
using System;
using System.Collections.Generic;

public static class testSimpleReflectionUserExtentions
{
   private static Dictionary<string, Type> properties = new Dictionary<string, Type>
       {
           { "Id", typeof(System.Guid)},
           { "FirstName", typeof(string)},
           { "LastName", typeof(string)},
       };

   public static Dictionary<string, Type> GetBakedType(this global::testSimpleReflection.User value)
   {
       return properties;
   }
}
```

Nothing supernature or complicated, but you have to implemant it for all types and it's not very pleasant job. We can ask C# compiler for some help. Rolyn analyzer can analyze our codebase and change it. So lets build a analyzer that is going to search for calls of our method ``` GetBakedType ``` and implement it for each required type.

To add support for such analyzer you will need just to install a one nuget package. Then we can just call our ``` GetBakedType ```, get compilation error, because it doesn't exist yet, calling code fix and it's done. We have implemented extension method thas is going to return all public properties assosiated with paticular type.

I think it will be easier to undestand what is happening in motion.

https://www.youtube.com/oembed?url=https%3A%2F%2Fwww.youtube.com%2Fwatch%3Fv%3DC9O1413fHac&format=json

If someone want to try it localy you can install nuget package SimpleReflection.

```
Install-Package SimpleReflection
```

Source code is [here]()

# Prepearing


# Implementation

I will not write about whole analyzer developing story, because it's relativly easy. Iam gonig describe only major steps.




