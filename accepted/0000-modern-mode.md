- Start Date: 2019-05-16
- RFC PR: (leave this empty)
- Umi Issue: (leave this empty)

# Summary

Umi Offers a "Modern Mode" to help someone to  bundle a modern bundle targeting modern browsers that support ES modules.

# Motivation

With Babel we are able to leverage all the newest language features in ES2015+, but that also means we have to ship transpiled and polyfilled bundles in order to support older browsers. These transpiled bundles are often more verbose than the original native ES2015+ code, and also parse and run slower. Given that today a good majority of the modern browsers have decent support for native ES2015, it is a waste that we have to ship heavier and less efficient code to those browsers just because we have to support older ones.

# Detailed design

Umi will produce two versions of your app: one modern bundle targeting modern browsers that support ES modules, and one legacy bundle targeting older browsers that do not.

# How We Teach This

Introduce a new configuration: modern

# Drawbacks

none

# Alternatives

none

# Unresolved questions

Later version safari still load nomodule scirpts. 
Ref: 
https://gist.github.com/samthor/64b114e4a4f539915a95b91ffd340acc
https://github.com/shaodahong/dahong/issues/18