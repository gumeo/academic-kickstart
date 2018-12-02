+++
title = "Part 2: Deep Learning with Closures in R"
date = 2017-12-21T13:40:09+02:00
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

# Let's go on!

If you just arrived, you can check out the first part [here](https://gumeo.github.io/post/part-1-deep-learning-with-closures-in-r/). The goal of this series is to demonstrate how compactly we can implement an MLP in a functional programming paradigm and how easy it becomes to extend/play around with it. This post is aimed at the R person that wants to get into deep learning or anyone curious about how these things are implemented. Another goal is to visualize what is happening in a neural network during training in hope to get a deeper understanding of what is going on. I posted [this gif](https://imgur.com/EgcQgkh) on the subreddit MachineLearning and dataisbeautiful, I got some constructive criticism on the gif and below is a *better* version where I have sped it up and removed the flickering and enhanced the contrast.

<center>
<blockquote class="imgur-embed-pub" lang="en" data-id="a/wIytV"><a href="//imgur.com/wIytV"></a></blockquote><script async src="//s.imgur.com/min/embed.js" charset="utf-8"></script>
</center>

So last time we got away with less than 50 lines of code to create a [factory](https://en.wikipedia.org/wiki/Factory_(object-oriented_programming)) for generating fully connected layers. This included a closure for a matrix class, (for making the layer have a mutable state), and an implementation of a forward pass. But the forward pass is basically a wrapper for updating the internals of the layer, we need to implement a forward pass for the whole network, and the same goes for the backwards propagation. This requires creating a function factory that constructs the network environment. This environment needs to have functions that implement forward-, backwards passes and a function for training. The following is a demonstration of such a function:

```{R, eval=FALSE}
# The following creates an MLP environment
# structNet is a vector defining the number of nodes in each layer, ignoring biases.
mlp <- function(structNet, minibatchSize,activation, initPos =FALSE, initScale=100){
  num_layers <- length(structNet)
  # Create the network
  layers <- list()
  for(i in 1:num_layers){
    if(i == 1){ # Input layer
      layers[[i]] <- Layer(activation, 
                           minibatchSize, 
                           c(structNet[1]+1,structNet[2]),
                           is_input=TRUE,
                           initPos = initPos,
                           initScale=initScale)
    }else if(i == length(structNet)){ # Output layer
      layers[[i]] <- Layer(softmax, 
                           minibatchSize, 
                           c(structNet[num_layers],structNet[num_layers]),
                           is_output=TRUE,
                           initPos = initPos,
                           initScale=initScale)
    }else{ # Hidden layers
      layers[[i]] <- Layer(activation, 
                           minibatchSize, 
                           c(structNet[i]+1,structNet[i+1]),
                           initPos = initPos,
                           initScale=initScale)
    }
  }
  
  forward_prop <- function(dataBatch){
    # Add bias to the input
    layers[[1]]$Z$setter(cbind(dataBatch,rep(1,nrow(dataBatch))))
    for(i in 1:(num_layers-1)){
      layers[[i+1]]$S$setter(layers[[i]]$forward_propagate())
    }
    return(layers[[num_layers]]$forward_propagate())
  }
  
  backwards_prop <- function(yhat,labels){
    layers[[num_layers]]$D$setter(t(yhat-labels))
    for(i in (num_layers-1):2){
      W_nobias <- layers[[i]]$W$getter()
      W_nobias <- W_nobias[1:(nrow(W_nobias)-1),]
      mat <- layers[[i]]$Fp$getter()
      layers[[i]]$D$setter((W_nobias%*%layers[[i+1]]$D$getter())*mat)
    }
  }
  
  update_weights <- function(eta){
    for(i in 1:(num_layers-1)){
      W_grad <- -eta*t(layers[[i+1]]$D$getter()%*%layers[[i]]$Z$getter())
      layers[[i]]$W$setter(layers[[i]]$W$getter()+W_grad)
    }
  }
  
  # Labels here as dummy matrix
  train <- function(trainData, trainLabels, num_epochs, eta, cvSplit = 0.3){
    # Code for preparing input
    for(i in 1:num_epochs){
      # Loop over epochs
      for(j in 1:numIter){
        # Loop over all minibatches
        # ...
        # Essential part for training
        preds <- forward_prop(tDat)
        backwards_prop(preds,tLab)
        update_weights(eta = eta)
      }
    }
  }
  
  myEnv <- list2env(list(network=layers,
                         forward_prop=forward_prop,
                         train = train))
  class(myEnv) <- 'mlp'
  return(myEnv)
}
```

The `mlp` function above starts by constructing the network given the layer configuration that we desire. Most of the implementation of the `train` function is omitted to highlight the essential part. The shown part highlights what is needed for stochastic gradient descent. We need to **forward propagate** the data, then **back propagate** the error, and finally based on this error we can **update the weights**. This is the heart and soul of SGD. Instead of using the whole dataset to estimate the gradient, we subsample a *minibatch* and estimate the gradient as an average over these samples. Quite a lot of gradient descent based optimization is done as a full batch optimization, specifically when people first learn about it, because SGD just adds an extra layer of complication. SGD allows for online updates, as more data arrives, and it may be computationally more efficient. The only issue is of course the diffculties of scaling the gradient, i.e. choosing the best learning rate, which is also an issue for regular/full battch gradient descent.

So the training function is essentially broken up into 3 steps, these steps are abstracted from the training algorithm, so we could easily change the `update_weights` function to include a momentum term or other things we might want to try, without changing the `train` function.

Notice that we take the vector `structNet` as input. This design can be extended to take in another vector describing what kind of layers we want and build the network with these different layers. This would generalize the implementation quite significantly. This would of course also require us to change the definition of the `Layer` function, or implement new layer functions.

# Is deep learning simple?

Now we have an `mlp` implementation in roughly 100 lines of code. I think this demonstrates in some way how *simple* an mlp really is, but also in contrast the difficulties needed to get to the modern implementations of deep learning. Modern implementations are much more general, where they usually implement a computation graph where all computations can be automatically differentiated. These are also made to target GPU hardware, where matrix-matrix and matrix-vector multiplication can be significantly accelerated. There is a package for R called `gpuR`, where one can do matrix calculations on the GPU. I might try that in the future, to see if I can speed this up significantly with minimal change of this code.

# How to use this?

First we need to define the activation function we want, the softmax function and a function to create a dummy encoding of factors variables in R. I have now included all of these in the package `minstr` available [here](https://github.com/gumeo/mnistr). So you do not need to define the yourself. Although you can of course define your own activation function as you wish. Take a look at the following to get an idea of how it is done.

```{R, eval=FALSE}
# rectified linear unit activation
# activation functions need a boolean parameter for 
# calculating the derivative
reLU <- function(X,deriv=FALSE){
  if(deriv==FALSE){
    X[X<0] <- 0 
    return(X)
  }else{
    X[X<0] <- 0
    X[X>0] <- 1
    return(X)
  }
}

# softmax for output layer
softmax <- function(X){
  Z <- rowSums(exp(X))
  X <- exp(t(X))%*%diag(1/Z)
  return(t(X))
}

# Function to represent factor as dummy matrix
class.ind <- function(cl) {
  Ik=diag(length(levels(cl)))
  x=Ik[as.numeric(cl),]
  dimnames(x) <- list(names(cl), levels(cl))
  x
}
```

Here is an example of how we can train a network with this. Feel free to play around with some of the parameters, such as the learning rate. If it is too big, nothing will be learned. The current configuration for the network works fine, but you can also try to squeeze it down or remove/add more hidden layers.

```{R, eval=FALSE}
# Load mnist dataset, note that you need to download it first
# mnistr::download_mnist() # This saves to the current working directory
mnistr::load_mnist()

# Visualize random digit, just to get familiar with the data, in case this is the 
# first time you see mnist
randDig <- sample(1:60000,1)
# Base R, notice that the data is in trainSet$x where each digit is stored as a row vector
image(matrix(trainSet$x[randDig,],28,28)[,c(28:1),drop=FALSE],main=paste(trainSet$y[randDig]),asp=1)
# ggplot function from the mnistr package, we can supply the data as a row vector
mnistr::ggDigit(trainSet$x[randDig,])

# Construct a network with two hidden layers (100 and 50 units) and rectified linear units
mnw <- mlp(structNet = c(784,100,50,10), 
           minibatchSize = 100, 
           activation = reLU,
           initPos = TRUE,
           initScale = 100)

# Define the training set
trainX <- trainSet$x
trainY <- class.ind(factor(trainSet$y))

# Call train
mnw$train(trainData = trainX/256,
          trainLabels = trainY,
          num_epochs = 3,
          eta = 0.005, 
          cvSplit = 0.3)
```


# No hidden layers

I wanted to visualize the training when there were no hidden layers. The weights in the gif below correspond to the training over one epoch, where the weights according to the digits are aligned like the matrix below:
$$
  \begin{bmatrix}
    2 & 5 & 8\\
    1 & 4 & 7\\
    0 & 3 & 6
  \end{bmatrix}
$$

<center>
<blockquote class="imgur-embed-pub" lang="en" data-id="a/s29MY"><a href="//imgur.com/s29MY"></a></blockquote><script async src="//s.imgur.com/min/embed.js" charset="utf-8"></script>
</center>

You can clearly see that the weights resemble templates that evolve to capture more and more of the variation present for each digit. The template also counts for making the patch dissimilar to the digits that it is not trying to match. This simple linear approach achieves a little more than 90% accuracy, which is amazing for how simple the method is. But still, for reading digits on checks, having every tenth read wrong is far from acceptable.

# Decomposing the network with magrittr

Now, one of the main things I wanted to achieve with this implementation was to decompose the network into it's individual components, to demonstrate how simple it truly is. Now I can in a simple manner compose functions that I have trained in order to produce some output. The `magrittr` is perfect for this, because then we do not need to nest inside parenthesis the composition of functions, we can just pipe the input through all the layers.

```{R, eval=FALSE}
# Needed for pipes
library(magrittr)

# Get the weight matricies from the network we trained above
w1 <- mnw$network[[1]]$W$getter()
w2 <- mnw$network[[2]]$W$getter()
w3 <- mnw$network[[3]]$W$getter()

set.seed(12345)
randDig <- sample(1:60000,1) # sample random digit
testDigit <- matrix(trainSet$x[randDig,]/256,nrow=1,ncol=28*28)
# Check label, it is a 2 for this seed
trainSet$y[randDig]
mnistr::ggDigit(testDigit)

# Add the bias term
testDigit <- cbind(testDigit,matrix(1,1,1))

# We are passing a single instance
simple_softmax <- function(X){
  Z <- sum(exp(X))
  X <- exp(X)/Z
  return(X)
}

# Now pipe it throught the network:
# Multiply by weights, use activation and then add 1 for bias
testDigit %>% 
  multiply_by_matrix(y=w1) %>% reLU %>% cbind(.,matrix(1,1,1)) %>% 
  multiply_by_matrix(y=w2) %>% reLU %>% cbind(.,matrix(1,1,1)) %>%
  multiply_by_matrix(y=w3) %>% simple_softmax %>% barplot
```

The network I trained is 99.6% certain that this is a two! The `magrittr` package allows us to write out the calculations happening inside the MLP in a very understandable way. Note that some networks used today have more than 50 layers, then this is not so useful anymore. In the case of 50plus layers we need other tricks to train it, e.g. skip connections and other optimizers.

# Code for generating the gifs

If you want to create you own gifs, it is probably easiest to define your own `mlp` function, just look at the `mnistr` implementation. Then add something similar to the following where the main update is happening. You could of course also save the weights to a matrix outside the environment using the `<<-` scope assignment operator.

```{R, eval=FALSE}
        # Where we do the SGD steps in the train function part of mlp
        preds <- forward_prop(tDat)
        backwards_prop(preds,tLab)
        update_weights(eta = eta)
        # The following is code for generating plots
        layerMat <- layers[[1]]$W$getter()
        # Removw bias
        layerMat <- layerMat[-nrow(layerMat),]
        # Set the number of columns and rows for the image
        nr <- 10
        nc <- 10
        weightsIm <- matrix(0,28*nr,28*nc)
        w <- 1
        # Fill the image with the relevant weights
        for(i in 1:nr){
          for(m in 1:nc){
            weightsIm[((i-1)*28+1):(i*28),((m-1)*28+1):(m*28)] <- matrix(layerMat[,w],28,28)[,c(28:1),drop=FALSE]
            w <- w + 1
          }
        }
        # Save for every fourth minibatch
        if(j%%4 == 1){
          data <- c(weightsIm)
          # Standardize to remove flickering
          nData <- qnorm(seq(0,1,len=28*28*nr*nc))[rank(data)]
          nIm <- matrix(nData,28*nr,28*nc)
          png(paste0('./plots/','reLU',sprintf("%08d", k),'.png'),width=28*nc*2,height=28*nr*2)
          # Remove margins so it looks nicer
          par(mar = rep(0, 4))
          image(nIm,asp=1,col=gray(seq(0,1,by=0.005),1),axes=FALSE)
          dev.off()
          k <- k+1
        }
```

# Next post

For the next post I plan to compare different activation functions and implement dropout for the network. It will probably be the final post until I get new ideas for stuff to implement.

<a href="https://twitter.com/share?ref_src=twsrc%5Etfw" class="twitter-share-button" data-size="large" data-text="Learning Deep Learning in R" data-url="https://gumeo.github.io/post/part-2-deep-learning-with-closures-in-r/" data-via="gumgumeo" data-hashtags="#rstats #machinelearning #deeplearning" data-lang="en" data-show-count="false">Tweet</a><script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
