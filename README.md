# tc39-proposal-object-gather
## Proposal: Object gather in parameters lists

This is a proposal to allow gathering one ore more __specified__ arguments into one or more objects when a function is called.
It's possible to gather some or all arguments into an array because there is no need to provide a key for each argument: array are objects that use numeric values as keys. To enable sort of object gather in parameters lists is mandatory to provide keys.


## Proposal's details
### The object gather
This syntax use the well know gather/rest operator `...` in conjunction with the block syntax `{}` cointaining a key list - that acts also as parameter list - to directly create a non constant object (identifiers in a normal parameter list are never constant) that gather all the parameters present into its list.
```js
function F(...obj{par1, par2}) {
  obj; // {par1: .., par2: ..}
  
  arguments[0] == obj.par1; // true
  arguments[1] == obj.par2; // true
}
```
It might be said that the object is an __implicit__ parameter. On the other hand the key list contains all the __explicit__ parameters that have to be passed and will populate the `arguments` array-like.
That function will be called as we call a function that use the array rest. This is one strength of the proposal because the call site remains clean:
```js
F(arg1, arg2);
```
\
Of course if we supply less arguments than requested, corresponding props will be undefined:
```js
function F(...obj{par1, par2, par3}) {
  obj; // {par1: .., par2: .., par3: ..}
}
```
```js
F(arg1); // 'obj.par2' and 'obj.par3' will be undefined
```
&nbsp;
### The object gather + array rest
We can still use the array rest operator in multiple ways:
1. After the object gather
```js
function F(...obj{par1, par2}, ...array) {
  obj; // {par1: .., par2: ..}
  array; // [.., .., .., ...]
  
  arguments[0] == obj.par1; // true
  arguments[1] == obj.par2; // true
  arguments[2] == array[0]; // true
}
```
Here the function call:
```js
F(arg1, arg2, arg3, arg4, arg5); 
// 'obj' will gather arg1 and arg2
// 'array' will gather or the other parameters
```

__But not before the object gather__:
```js
function F(...array, ...obj{par1, par2}) {
    // Error: obj couldn't be populated
}
```

2. Inside the object gather __at the bottom__ of the key list
```js
function F(...obj{par1, par2, ...array}) {
  obj; // {par1: .., par2: .., array: [.., .., .., ...]}
  
  arguments[0] == obj.par1; // true
  arguments[1] == obj.par2; // true
  arguments[2] == obj.array[0]; // true
}
```
Here the function call:
```js
F(arg1, arg2, arg3, arg4, arg5); 
// 'obj' will gather arg1 and arg2 directly and all the other params into its 'array' prop
```

__We cannot add any parameters in the function signature if we use the array rest inside an object gather:__
```js
function F(...obj{par1, par2, ...array}, ...array2) {
    // Error: array2 couldn't be populated
}
```
&nbsp;
### The object gather + object gather
More than one _object gather_ could be used toghether, mainly in two ways:
1. Sibling object gathers
```js
function F(...obj{par1, par2}, ...obj2{par3, par4}) {
  obj; // {par1: .., par2: ..}
  obj2; // {par3: .., par4: ..}
  
  arguments[0] == obj.par1; // true
  arguments[1] == obj.par2; // true
  arguments[2] == obj2.par3; // true
  arguments[3] == obj2.par4; // true
  arguments[4] == void 0; // true
}
```
Here the function call:
```js
F(arg1, arg2, arg3, arg4); 
// 'obj' will gather arg1 and arg2
// 'obj2' will gather arg3 and arg4
```

2. Nested object gathers
```js
function F(...obj{par1, ...obj2{par2, par3}, par4}) {
  obj; // {par1: .., obj2: {par2: .., par3: ..}, par4: .. }
  
  arguments[0] == obj.par1; // true
  arguments[1] == obj.obj2.par2; // true
  arguments[2] == obj.obj2.par3; // true
  arguments[3] == obj.par4; // true
  arguments[4] == void 0; // true
}
```
Here the function call:
```js
F(arg1, arg2, arg3, arg4); 
// 'obj' will gather arg1 and arg4 directly
// 'obj' will gather arg2 and arg3 into its 'obj2' object prop
```
&nbsp;
### Special case: this object gather
In constructor functions, and in the `constructor` method of classes, the object gather syntax could be useful to quickly assign all parameters to the `this` object:
```js
function F(...this{par1, par2, par3}) {
  this; // {par1: .., par2: .., par3: ..}
  
  arguments[0] == this.par1; // true
  arguments[1] == this.par2; // true
  arguments[2] == this.par3; // true
  arguments[3] == void 0; // true
}
```
Here the weel know function call:
```js
new F(arg1, arg2, arg3); 
// 'this' will gather all the arguments
```
\
__We cannot use this syntax in subclasses because of the prior super() call:__
```js
constructor(...this{par1, par2, par3}) {
  //  ReferenceError: must call super constructor before using |this| in ... class constructor
}
```


## Reasons
* Currently this type of gather is not allowed in Javascript.
* It provides a fast and unequivocal way to construct object using parameters that could be easily optimized by the engine.
* It provides a fast and clean way to create common constructor functions. 
* Current solutions imply one or more of the following: changes to the function signature, useless objects creations, annoying identifiers repetitions. Let's briefly see them:


### annoying identifiers repetitions
To not change the function signature:
```js
function F(par1, par2, par3, ..., parN){
  // function body
}
```
to gather arguments into an object we fall in the case of the annoying identifiers repetitions. This is not DRY:
```js
function F(par1, par2, par3, ..., parN){
  const obj = { par1, par2, par3, ..., parN }
}
```

### changes to the function signature
To avoid above repetitions we are forced to change the function signature (and this is something I'd like to avoid):
```js
function F(obj){
  // function body
}
```
calling the function in this way:
```js
F({arg1, arg2, arg3, ..., argN});
```
It could seem a minor issue.
But why if the object is a construct necessary only for function's internal needs have we to create it in the call site? 

### annoying identifiers repetitions || useless objects creations
Them are evident when we are in a constructor function:
```js
function constructor(par1, par2, par3, ..., parN){
  this.par1 = par1;
  this.par2 = par2;
  this.par3 = par3;
  ...
  this.parN = parN;
}
```
This is annoying and obviously not a DRY approach.
We could have done like this:
```js
function constructor(par1, par2, par3, ..., parN){
  Object.assign(this, {par1, par2, par3, ..., parN});
}
```
Here we are creating and destroying in a short time an object, __copying__ each value two times: from the parameters to the object, then from the object to `this`. It is also present a function call to `Object.assign`.

## How could the syntax be transpiled?
For the sake of argument we can assert that Babel could not follow a DRY approach transpiling our code, giving priority to a faster and less memory needy solution. Probably the solutions tagged with _annoying identifiers repetitions_ are the best:

### normal object gather
```js
function F(...obj1{par1a, par1b}, ...obj2{par2a, par2b}) {
    // function body
}
```
could be transpiled into:
```js
function F(par1a, par1b, par2a, par2b) {
    let obj1 = {par1a, par1b};
    let obj2 = {par2a, par2b};
    // function body
}
```
Parameters could be reassigned so we use `let` instead of `const`.

### nested object gather
```js
function F(...obj{par1, ...obj2{par2, par3}, par4}) {
    // function body
}
```
could be transpiled into:
```js
function F(par1, par2, par3, par4) {
    let obj1 = {par1, obj2: {par2, par3}, par4};
    // function body
}
```

### this gather
```js
function F(...this{par1, par2, par3}) {
    // function body
}
```
could be transpiled into:
```js
function F(par1, par2, par3) {
    this.par1 = par1;
    this.par2 = par2;
    this.par3 = par3;
}
```
