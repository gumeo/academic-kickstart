+++
title = "Part 1: Deep Learning with Closures in R"
date = 2017-12-14T13:40:09+02:00
draft = false

# Tags and categories
# For example, use `tags = []` for no tags, or the form `tags = ["A Tag", "Another Tag"]` for one or more tags.
tags = []
categories = []

# Featured image
# Place your image in the `static/img/` folder and reference its filename below, e.g. `image = "example.jpg"`.
# Use `caption` to display an image caption.
#   Markdown linking is allowed, e.g. `caption = "[Image credit](http://example.org)"`.
# Set `preview` to `false` to disable the thumbnail in listings.
[header]
image = ""
caption = ""
preview = true

+++

# Start of a small series

The gif below is the evolution of the weights from a neural network trained on the mnist dataset. Mnist is a dataset of handwritten digits, and is kind of the **hello world/FizzBuzz** of machine learning. It is maybe not a very challenging data set, but you can learn a lot from it. This is a 10 by 10 grid of images, where each individual small patch represents weights going to a single neuron in the first hidden layer of the network. After I saw the content by Grant Sanderson, I wanted to inspect by myself what the network is actually learning. This series is going to outline this development, with an angle towards functional programming.

<center>
<blockquote class="imgur-embed-pub" lang="en" data-id="EgcQgkh"><a href="//imgur.com/EgcQgkh"></a></blockquote><script async src="//s.imgur.com/min/embed.js" charset="utf-8"></script>
</center>

So I received the [Deep Learning book](http://www.deeplearningbook.org/) a little more than a month ago, and I have had time to read parts I and II. I think that overall the authors successfully explain and condense a lot of research into something digestable. The reason why I use the word condense is because how much information the book contains. The bibliography is 55 pages, so I almost feel that the book should be called *Introduction to Deep Learning Research*, because it is a gateway to so much good material. Another fascinating thing about this book is the discussion it contains. The authors are quite upfront about some criticisms on deep learning and discuss them to a great extent. All in all I look forward to finish reading the book.

I have fiddled around with deep learning since I took a summer-school on it in 2014, where [Hugo Larochelle](https://twitter.com/hugo_larochelle?ref_src=twsrc%5Egoogle%7Ctwcamp%5Eserp%7Ctwgr%5Eauthor) was giving lectures and he joined us for an epic sword fighting competition/LARP session in the countryside of Denmark. I have followed the evolution of deep learning since, and what has been most amazing is by how much the barrier of entry has been lowered. The current software frameworks make it so easy to get started. After reading the DL book I found a strong inner urge to implement some of these things myself. I think that a way to get a better understanding of programming concepts, algorithms and data structures, is just to go for it and implement it. Also talking about data-structures, deep learning is also becoming a part of that:

<center>
<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">Jeff Dean and co at GOOG just released a paper showing how machine-learned indexes can replace B-Trees, Hash Indexes, and Bloom Filters. Execute 3x faster than B-Trees, 10-100x less space. Executes on GPU, which are getting faster unlike CPU. Amazing.  <a href="https://t.co/PPVkrFVKXg">https://t.co/PPVkrFVKXg</a></p>&mdash; Nick Schrock (@schrockn) <a href="https://twitter.com/schrockn/status/940037656494317568?ref_src=twsrc%5Etfw">December 11, 2017</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
</center>

After seeing the videos made by [Grant Sanderson](http://www.3blue1brown.com/) on deep learning I decided it was time to implement the basics by myself. I wanted to completely understand back-propagation, how it really is a dynamic programming algorithm where we actually do some smart book-keeping and reuse computations. That is one of the major *tricks* that makes this algorithm work.

## Not another framework

But of course implementing a fully fledged DL framework is a feat one should not do, unless you have some very specific reason for it. Many frameworks have been created, and doing this alone **to learn** should not have that goal in mind. I wanted to make something that would be easily extendable, I also wanted to do it in R (because I really like R), I also made a package at some point called [mnistr](https://github.com/gumeo/mnistr) where I wanted to add some fun examples with neural networks on. When I'm done with this series I'll add the implementation to the package and submit it to CRAN.

## Closures

The final reason I had for doing this relates to closures. So deep learning, or deep neural networks, are in it's essance just functions, or rather compositions of functions. These individual functions are usually not that complicated, what makes them complicated is what they learn from complicated data, and how these individual simple things together make something complicated. We do not completely understand what they learn or do. If I can make a representation of a multi-layer percepteron (*which is a fully connected neural network*) in a functional programming paradigm, then it might be easier to understand it for others and myself. I will hopefully be able to disentanlge the networks into inidvidual functions and using the [magrittr](https://cran.r-project.org/web/packages/magrittr/index.html) package to do the function composition in a more obvious manner, which demonstrates that the individual pieces of a neural network are not so complicated, and the final composition does not need to be so complicated either. 

## But wait a minute? What are closures?

So the [wikipedia article](https://en.wikipedia.org/wiki/Closure_(computer_programming)) gives it a pretty good treatment

> In programming languages, closures (also lexical closures or function closures) are techniques for implementing lexically scoped name binding in languages with first-class functions. Operationally, a closure is a record storing a function[a] together with an environment:[1] a mapping associating each free variable of the function (variables that are used locally, but defined in an enclosing scope) with the value or reference to which the name was bound when the closure was created.[b] A closure—unlike a plain function—allows the function to access those captured variables through the closure's copies of their values or references, even when the function is invoked outside their scope

This sounds a lot like object oriented programming, but when the *objects* that we are working with, are naturally functions, then this works very nicely. But let's look at a simple example!

```r
newBankAccount <- function(){
  myMoney <- 0
  putMonyInTheBank <- function(amount){
    myMoney <<- myMoney + amount
  }
  howMuchDoIOwn <- function(){
    print(paste('You have:',myMoney,'bitcoins!'))
  }
  return(list2env(list(putMonyInTheBank=putMonyInTheBank,
                       howMuchDoIOwn=howMuchDoIOwn)))
}
```

Now I can use this to create a *bank account function*

```r
> myAccount <- newBankAccount()
> myAccount$howMuchDoIOwn()
[1] "You have: 0 bitcoins!"
> myAccount$putMonyInTheBank(200)
> myAccount$howMuchDoIOwn()
[1] "You have: 200 bitcoins!"
> copyAccount <- myAccount
> copyAccount$howMuchDoIOwn()
[1] "You have: 200 bitcoins!"
> copyAccount$putMonyInTheBank(300)
> copyAccount$howMuchDoIOwn()
[1] "You have: 500 bitcoins!"
> myAccount$howMuchDoIOwn() # Ahh snap, I also received bitcoins!
[1] "You have: 500 bitcoins!"
```

So compared to *normal* functions, now we have a function (*actually an environment*) with a mutable state. Now we can have side-effects when we call the functions from the environment. But if you look at the calls I made above, you may have noticed that when I copied the account, added to the copied account, the original account also had an increased amount in it. So the copied account didn't get the data copied, only the references. So if you make multiple bank accounts, initialize each seperately, don't initialize one prototype and copy it to all the users. Otherwise all the users end up with one big shared account!

So be careful, and if you really want to copy environments look at [this SO post](https://stackoverflow.com/questions/9965577/r-copy-move-one-environment-to-another). If you want to learn more about environments and the functional programming side of R, [advanced R](http://adv-r.had.co.nz/) by Hadley Wickham is a great starting point, you might also want to check out [this blogpost](http://www.lemnica.com/esotericR/Introducing-Closures/). Also if you have coded in javascript, you might be familiar with the issue of copying closures there. And btw, I have no bitcoins :(

Another important thing that you might have noticed is the assignment operators. If you are not familiar with R, `<-` is pretty much the same as `=`, they are kind of used interchangably, but they have a very subtle difference that you can read about [here](https://stackoverflow.com/questions/1741820/assignment-operators-in-r-and). The weird assignment operator that you noticed was probably the `<<-`, the scoping assignment. This is the whole trick about closures, best said [here](https://stackoverflow.com/questions/2628621/how-do-you-use-scoping-assignment-in-r?rq=1):

> A closure is a function written by another function. Closures are so called because they enclose the environment of the parent function, and can access all variables and parameters in that function. 

The scoping operator creates the reference needed, such that the returned function encloses the environment of the caller. This is why closures are sometimes called poor man's objects. We are basically emulating the creation of public and private members in some sense, but without the overhead of object oriented structured code. This is an essential part in making the implementation look cleaner and more transparent. The code that you write is more to the point of solving the task as hand, rather the adhearing to a structure of a particular paradigm.

## Too the point on deep learning again!

For the structure of the implementation I was very inspired by [this post](http://briandolhansky.com/blog/2014/10/30/artificial-neural-networks-matrix-form-part-5) by Brian Dolhansky, where he implements an MLP in python.

This structure completely encapsulates what is needed in a layer and how things are connected. Instead of creating a class, I am going to make a closure. So when I say a **layer**, I mean the activations from the previous layers and the outgoing weights. So the only layer that doesn't have, or doesn't need weights, is the output layer.

This closure is therefore a function, or has some functions, which makes sense for a layer in a neural network, which is essentially a function in the mathematical sense. But before we get to the layer, we need what essentially creates the closure, a building block for a matrix:

```r
matrixInLayer <- function(init = FALSE, rS, cS, initPos = FALSE, initScale = 100){
  intMat <- matrix(0, nrow=rS, ncol=cS)
  if(init == TRUE){
    intMat <- matrix(rnorm(rS*cS)/initScale,nrow = rS,ncol = cS)
    if(initPos == TRUE){
      intMat <- abs(intMat)
    }
  }
  getter <- function(){
    return(intMat)
  }
  setter <- function(nMat){
    intMat <<- nMat
  }
  return(list2env(list(setter = setter, getter=getter)))
}
```

We parameterize this function such that we can account for different initialization strategies in the weights, but we can use this to create all the needed matricies in a layer. The layer closure is then just something that encapsulates the internal state of the network, with placeholders for all the data needed to do a forward and backwards pass. The essential trick to make this work is of course the scope assignment in the `setter` function. The fully connected layer can now be implemented as:

```r
Layer <- function(activation, minibatchSize, sizeP, is_input=FALSE, is_output=FALSE, initPos, initScale){
  # Matrix holding the output values
  Z <- matrixInLayer(FALSE,minibatchSize,sizeP[1])
  # Outgoing Weights
  W <- matrixInLayer(TRUE,sizeP[1],sizeP[2],initPos=initPos, initScale=initScale)
  # Input to this layer
  S <- matrixInLayer(FALSE,minibatchSize,sizeP[1])
  # Deltas for this layer
  D <- matrixInLayer(FALSE,minibatchSize,sizeP[1])
  # Matrix holding derivatives of the activation function
  Fp <- matrixInLayer(FALSE,sizeP[1],minibatchSize)
  # Propagate minibatch through this layer
  forward_propagate <- function(){
    if(is_input == TRUE){
      return(Z$getter()%*%W$getter())
    }
    Z$setter(activation(S$getter()))
    if(is_output == TRUE){
      return(Z$getter())
    }else{
      # Add bias for the hidden layer
      Z$setter(cbind(Z$getter(),rep(1,nrow(Z$getter()))))
      # Calculate the derivative
      Fp$setter(t(activation(S$getter(),deriv = TRUE))) 
      return(Z$getter()%*%W$getter())
    }
  }
  # Return a list of these functions
  myEnv <- list2env(list(forward_propagate=forward_propagate, S = S,
                         D = D, Fp = Fp, W = W, Z = Z))
  class(myEnv) <- 'Layer'
  return(myEnv)
}
```

The layer function includes all the things needed for doing the book-keeping of the calculations for backpropagation. With very little extra code we have the ability to have arbitrary minibatch sizes and arbitrary activation functions. The activation function just needs and extra boolean parameter to determine whether we are evaluating the function or calculating the deivative. (*I'll go in more detail in the next post about what a minibatch is when I cover stocastic gradient descent*). We can protype these activation functions on the fly, because R is a functional language. So in less than 50 lines of code we have already created the heart of a multilayer percepteron, with some generalizability. So what does this layer do? It takes input from the activations of the neurons in the previous layer as a vector, multiplies it with a matrix and element-wise applies the activation. In essance it is this:
$$
  \sigma(\mathbf{W}\cdot \mathbf{x})
$$
where $\mathbf{W}$ is the weight matrix, $\mathbf{x}$ is the input and $\sigma$ is the activation function. The problem of deep learning is then to find good parameters for the weights $\mathbf{W}$ when these functions are composed.

## Next post

This ended up being a lot longer than I anticipated, but for the next post I aim at finishing the implementation for the MLP, going through backpropagation and simple training on mnist. For the last post the goal is to show how something like dropout can very easily be incorporated in this implementation and how we can disentangle a network using the magrittr package. Implementing other kinds of layers and different optimization strategies is also on the drawing board.

If you like this post I would greatly appreciate a tweet, my twitter handle is @gumgumeo :)

<div class="center">
<a href="https://twitter.com/share?ref_src=twsrc%5Etfw" class="twitter-share-button" data-size="large" data-text="Learning Deep Learning in R" data-url="https://gumeo.github.io/post/part-1-deep-learning-with-closures-in-r/" data-via="gumgumeo" data-hashtags="#rstats #machinelearning #deeplearning" data-lang="en" data-show-count="false">Tweet</a><script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
</div>
