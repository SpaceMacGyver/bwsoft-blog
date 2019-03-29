---
title: Tacit programming in JavaScript
date: 2019-03-29T15:37:10.792Z
description: 'A point-free, but not pointless, exercise in making code readable'
---
Prior to getting a job a while back, I was given an informal phone-based interview by another developer, with the intent of sizing me up to see if I was everything my resume said I was.  It was pretty laid, back, with the interviewer asking questions that probed the various claims I had made.  Among the things I was probed on was my opinion on comments in code.  I proudly declared my love for them, at which point I could tell the programmer on the other end of the phone didn't see eye-to-eye with me.  This didn't stop me from getting the job, but nagged at me somewhat.  "Why wouldn't someone like _more_ explanation from their code?"

After landing that job and spending some time in that ecosystem, I came to understand something.  Good code doesn't require explanation, because it's **self-explanatory**.  Good code is expressive in all the right ways, gently guiding a reader without sacrificing brevity or code structure.  For the sake of this discussion, I will qualify "good code" as code meeting many of [these ideals](https://gist.github.com/wojteklu/73c6914cc446146b8b533c0988cf8d29), originally espoused by "Uncle Bob" (Robert) Martin.

Among the ideas expressed therein, there are some that are relevant to this discussion:
* Use descriptive function names
* Functions should do one thing
* Always try to explain yourself in code
* Don't add obvious noise with comments

_(I highly recommend not only reading the referenced gist in its entirety, but also checking out the book from which the ideals are derived, [Clean Code](https://www.google.com/search?q=978-0132350884).)_

Putting these principles into practice leverages what I believe to be the greatest skill a programmer can possess: the ability to reduce a large problem into smaller problems.  Consider the following function:

```js
// NOTE: Why are we doing this again (2019-02-04 RJ)
function doTheThing(x) {
    let y = [];
    let z = 0;

    // Do it for all the things:
    for (var i in x) {
        var r = x[i] * 2;

        // It goes at the end:
        y.push(r);

        if (r > z) {
            z = r;
        }
    }

    return [y, z];    
}
```

I'd invite you to stare at it for a while and see what it does, but in a perfect world you wouldn't need to even if it was complex or solving a hard problem in the first place.  I'll do you a favor and fix one of the glaring problems, the poorly chosen names of variables and functions:

```js
// NOTE: Why are we doing this again (2019-02-04 RJ)
function doubleItemsAndFindLargest(listOfNumbers) {
    let doubledList = [];
    let largestItem = 0;

    // Do it for all the things:
    for (var key in listOfNumbers) {
        var currentItem = listOfNumbers[key] * 2;

        // It goes at the end:
        doubledList.push(currentItem);

        if (currentItem > largestItem) {
            largestItem = currentItem;
        }
    }

    return [doubledList, largestItem];
}
```

I've done nothing to the logic or the comments, but suddenly it becomes immediately apparent what we're trying to do.

Also, it becomes apparent that this function breaks one of the highlighted rules: that functions should only do one thing.  This one is doing two things, doubling and finding the maximum value.  Since fixing this issue requires addressing the consuming end, I'll just focus on the changes that can improve this function.

Looking closely at the comments, I realize I gain nothing of value from them.  I think that we've already answered the question posed in the first comment by giving the function a descriptive name, so I'll nix that one.  The second comment tells us nothing that the code itself doesn't (a `for` statement will naturally do what it does best, iterate over all of the things), and similarly the next comment is simply telling the programmer something they should already know about `Array.prototype.push`.

```js
function doubleItemsAndFindLargest(listOfNumbers) {
    let doubledList = [];
    let largestItem = 0;

    for (var key in listOfNumbers) {
        var currentItem = listOfNumbers[key] * 2;

        doubledList.push(currentItem);

        if (currentItem > largestItem) {
            largestItem = currentItem;
        }
    }

    return [doubledList, largestItem];
}
```

From a black-box perspective, this function is pure, in that it takes in a value and produces a new value without affecting the input, and does not depend on any external dependencies.  Internally, though, there is considerable mutation:

* `doubledList` is appended to for every item that is contained in `listOfNumbers`
* `largestItem` is redefined whenever a number larger than its current value is encountered

Although the two mutations are taking place in the same loop, they could just as easily be represented with two separate operations.  For the moment, I'm going to create two separate loops to illustrate the point, adding relevant comments:

```js
function doubleItemsAndFindLargest(listOfNumbers) {
    let doubledList = [];
    let largestItem = 0;

    // Double listOfNumbers elements and assign to doubledList
    for (var key in listOfNumbers) {
        var currentItem = listOfNumbers[key] * 2;

        doubledList.push(currentItem);
    }

    // Assign the max value from doubledList to largestItem
    for (var key in doubledList) {
        var currentItem = doubledList[key];

        if (currentItem > largestItem) {
            largestItem = currentItem;
        }
    }

    return [doubledList, largestItem];
}
```

What makes these comments different from their predecessors is that they offer clarity.  However, they would be unnecessary if we extracted dense logic into separate functions:

```js
function doubleListOfNumbers(list) {
    let returnValue = [];

    for (var key in list) {
        var currentItem = list[key] * 2;

        returnValue.push(currentItem);
    }

    return returnValue;
}

function findLargestItem(list) {
    let largestItem = 0;

    for (var key in list) {
        var currentItem = list[key];

        if (currentItem > largestItem) {
            largestItem = currentItem;
        }
    }

    return largestItem;
}

function doubleItemsAndFindLargest(listOfNumbers) {
    let doubledList = doubleListOfNumbers(listOfNumbers);
    let largestItem = findLargestItem(doubledList);

    return [doubledList, largestItem];
}
```

By extracting those two functions (`doubleListOfNumbers` and `findLargestItem`), I've exposed the two core behaviors of the `doubleItemsAndFindLargest` function.  I've now identified that the first of those two functions is essentially transforming the input list, which could just as easily be done with a call to `Array.prototype.map`:

```js
function findLargestItem(list) {
    let largestItem = 0;

    for (var key in list) {
        var currentItem = list[key];

        if (currentItem > largestItem) {
            largestItem = currentItem;
        }
    }

    return largestItem;
}

function doubleItemsAndFindLargest(listOfNumbers) {
    let doubledList = listOfNumbers.map(x => x * 2);
    let largestItem = findLargestItem(doubledList);

    return [doubledList, largestItem];
}
```

This might seem like a step backwards, since now we have the variable `x` that doesn't state its purpose, but I'll make a judgment call that any reader should quickly see that we have a doubling of all inputs.  If this becomes a stickling point we could always extract that behavior, but this seems like a reasonable compromise.

Likewise, I also see that `findLargestItem` could be replaced with a call to `Array.prototype.reduce`:

```js
function doubleItemsAndFindLargest(listOfNumbers) {
    let doubledList = listOfNumbers.map(x => x * 2);
    let largestItem = doubledList.reduce((x, y) => Math.max(x, y), 0);

    return [doubledList, largestItem];
}
```

Again, we're introducing single-letter variable names `x` and `y`, but for the same reason as above I'll let it lie.  The `0` at the end, though, is a [magic number](https://en.wikipedia.org/wiki/Magic_number_(programming)#Unnamed_numerical_constants), and it's not as obvious what it does.  I'll extract it to give it a name:

```js
const SMALLEST_POSSIBLE_VALUE = 0;

function doubleItemsAndFindLargest(listOfNumbers) {
    let doubledList = listOfNumbers.map(x => x * 2);
    let largestItem = doubledList.reduce((x, y) => Math.max(x, y), SMALLEST_POSSIBLE_VALUE);

    return [doubledList, largestItem];
}
```

Okay, so reading from top to bottom, `doubleItemsAndFindLargest` first doubles each item in its original list to create `doubledList`, and then traverses that list to find the largest (max) item.

In its current form, `doubleItemsAndFindLargest` no longer has the problem of internal mutation*.  This function's in a good place already, but I'm going to pull an extra trick out of my hat and employ a third-party library, Ramda, to remove the `let` statements entirely:

```js
// Assuming R (the Ramda library) is in scope:
const {compose, map, reduce, max} = R;
const SMALLEST_POSSIBLE_VALUE = 0;

const doubleItemsAndFindLargest = compose(
    list => [
        list,
        reduce(max, SMALLEST_POSSIBLE_VALUE, list)
    ],
    map(x => x * 2)
);
```

To read the above, `doubleItemsAndFindLargest` is now an operation where, reading from right-to-left, we first map over all items and double them (producing value `r1`), then we produce an array whose first value is the input `r1` ("identity"), and the second value is the result of reducing over all items in `r1` with the `max` function starting from the smallest possible value.

Admittedly this final form might be hard to read, but with some light refactoring we can name things better:

```js
// Assuming R (the Ramda library) is in scope:
const {compose, map, reduce, max} = R;
const SMALLEST_POSSIBLE_VALUE = 0;

const findLargestValue = reduce(max, SMALLEST_POSSIBLE_VALUE);
const returnInputAndLargestItem = list => [
    list,
    findLargestValue(list)
];
const doubleTheList = map(x => x * 2);

const doubleItemsAndFindLargest = compose(
    returnInputAndLargestItem,
    doubleTheList
);
```

Unfortunately, because we're technically returning two distinct items from our function, some complexity will remain regardless of how we approach the problem.  However, this final form illustrates some qualities that lend themselves well to both readability and reuse:

* Every operation is distilled into a small unit of computation.
* Complex operations are formed by [composing](https://en.wikipedia.org/wiki/Function_composition_(computer_science)) smaller operations.
* Operations do what they say, and say what they do, through good naming.
* Operations are pure; they always give the same output for the same inputs, and don't depend on globals or higher-scoped variables.

_*: Under the hood, `Array.prototype.reduce` actually does do some internal mutation, but this is an implementation detail that you as a consumer would not need to be aware of._
