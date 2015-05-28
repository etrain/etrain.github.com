---
layout: post
title: "Type Safe Linear Algebra in Scala"
description: ""
category: 
tags: []
---
{% include JB/setup %}
Thanks to Scala, lately I've been appreciating type safety more and more when programming. It was an adjustment coming from Python, R, and C, but the performance benefits and the fact that I can be pretty sure that when my code compiles, it will run properly means that I can deploy code with much higher confidence.

However, there's one area of my development life where type safety hasn't done much for me -- specifically numerical linear algebra and, by consequence, machine learning. In this post I'll explain what that problem is, and propose a solution to backport type safety onto linear algebra operations in Scala, in a non-intrusive way.

###The Problem

Anyone who has taken a basic linear algebra class or played around with numerical code knows about dimension alignment - in python it looks like this:
{% highlight python %}
>>> np.random.rand(2,2) * np.random.rand(3,1)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
ValueError: operands could not be broadcast together with shapes (2,2) (3,1) 
{% endhighlight %}

In Scala, using the *awesome* [breeze](https://github.com/scalanlp/breeze) library, it looks like this:
{% highlight scala %}
scala> import breeze.linalg._
import breeze.linalg._

scala> DenseMatrix.rand(2,2) * DenseMatrix.rand(3,1)
java.lang.IllegalArgumentException: requirement failed: Dimension mismatch!
	at scala.Predef$.require(Predef.scala:233)
	at breeze.linalg.operators.DenseMatrixMultiplyStuff$implOpMulMatrix_DMD_DMD_eq_DMD$.apply(DenseMatrixOps.scala:45)
	at breeze.linalg.operators.DenseMatrixMultiplyStuff$implOpMulMatrix_DMD_DMD_eq_DMD$.apply(DenseMatrixOps.scala:40)
	at breeze.linalg.NumericOps$class.$times(NumericOps.scala:261)
	at breeze.linalg.DenseMatrix.$times(DenseMatrix.scala:54)
...
{% endhighlight %}

That is - if you want to multiply two matrices, their dimensions have to match up in the right way. An (n x d) matrix can only be multiplied on the left by a matrix that's (something x n) or on the right by a matrix that's (d x something). 

There's something to notice about the errors above. First, they're data dependent. Multiplying a (3 x 2) by a (2 x 1) matrix wouldn't have such disastrous effects, but change the inner dimension, and suddenly you have problems. Second, they're *runtime* errors. Meaning that, especially in the case of Scala, the code will compile just fine, and we will only encounter this error at runtime. Isn't this what the compiler is supposed to figure out for us?

Matrix-matrix multiplication is at the very heart of advanced analytics, machine learning, and high performance scientific computing - so it's comes up in complicated and non-trivial ways at the center of some very complicated algorithms. I can't tell you the number of bugs I've hit because I forgot to transpose or because I assumed that the data was coming in in one shape and in fact it came in in another, and I believe this to be a common experience among programmers of algorithms like this. Heck - even the theoreticians will tell you that half the work in checking their proofs for correctness is making sure that the dimensions line up. (I kid, but only a little.)

###A Solution

So how do we avoid this mess and get the type system to check our dimensions for us? If you came to this post hoping to read about Algebraic Data Types and Monads, then I'm sorry, this is not the post you were hoping for. Instead, I'll propose a simple solution that does the trick without resorting to type system black magic. 

The basic observation here is twofold:
1. Usually people work with a relatively small number of dimensions. That is, I might have "N" data points with "D" features in "K" classes - while each of these numbers might itself be big, there are only 3 of them to keep track of, and I kind of know that my data is going to be (N x D) and my model is going to be (D x K), for example.
2. By forcing the user to provide just a little more information to the type system, we can get type safety for linear algebra in a sensible way.

So, now for the code - first, let's define a Matrix type that contains two type parameters - A and B, which has some basic operations:
{% highlight scala %}
import breeze.linalg._

class Matrix[A,B](val mat: DenseMatrix[Double]) {
    def *[C](other: Matrix[B,C]): Matrix[A,C] = new Matrix[A,C](mat*other.mat)
    def t: Matrix[B,A] = new Matrix[B,A](mat.t)
    def +(other: Matrix[A,B]): Matrix[A,B] = new Matrix[A,B](mat + other.mat)
    def :*(other: Matrix[A,B]): Matrix[A,B] = new Matrix[A,B](mat :* other.mat)
    def *(scalar: Double): Matrix[A,B] = new Matrix[A,B](mat * scalar)
}
{% endhighlight %}

Additionally, I'll create some helper functions - one to read data in from file and the other to invert a square matrix:

{% highlight scala %}
object MatrixUtils {
  def readcsv[A,B](filename: String) = new Matrix[A,B](csvread(new java.io.File(filename)))
  
  def inverse[A](x: Matrix[A,A]): Matrix[A,A] = new Matrix[A,A](inv(x.mat))
  
  def ident[D](d: Int): Matrix[D,D] = new Matrix[D,D](DenseMatrix.eye(d))
  
}
{% endhighlight %}

So let's see it in action:
{% highlight scala %}
import MatrixUtils._

class N
class D
class K

val x = new Matrix[N,D](DenseMatrix.rand(100,10))
val y = new Matrix[N,K](DenseMatrix.rand(100,2))

val z1 = x * x //Does not compile!
val z2 = x.t * y //Compiles! Returns a Matrix[D,K]
val z3 = x.t * x //Compiles! Returns a Matrix[D,D]
val z4 = x * x.t //Compiles! Returns a Matrix[N,N]
{% endhighlight %}

What have we done her? We've first defined some classes to represent our dimensions (which are abstract) - then we've created some matrices and assigned labels to these dimensions. We could just has easily have read `x` or `y` from file - provided we knew their intended shapes.

Finally, we tried some basic linear algebra (matrix multiplication!) on this stuff. 

###So what?

Well, here's the punchline - we can now implement something reasonably complicated - say, solving an L2-regularized linear system using the normal equations - using the above classes, be sure that my code is actually going to run if I feed it data of the right shape, and as a bonus have the compiler confirm for me that my method actually has the right dimensions.

Suppose I want to find the solution to the following problem 

\\[ \underset{x}{min\,}{ {\\|A X - B\\|}_2^2 + \lambda \\|X\\|_2^2} \\]

A and B are fixed matrixes (say "data" and "labels" in the case of machine learning.) One way to do this is to take the derivative of the above (convex) expression and set it to 0. This results in the fairly complicated expression:

\\[ X = (A^T A + \lambda I)^{-1} A^T B \\]

Or, written with my handy Matrix library:

{% highlight scala %}
import MatrixUtils._

def solve[X,Y,Z](a: Matrix[X,Y], b: Matrix[X,Z], lambda: Double) = {
  inverse((a.t * a) + ident[Y](a.mat.cols)*lambda) * a.t * b
}
{% endhighlight %}

And what does the type signature of solve look like?

{% highlight scala %}
solve: [X, Y, Z](a: Matrix[X,Y], b: Matrix[X,Z], lambda: Double)Matrix[Y,Z]
{% endhighlight %}

The compiler has figured out that the result of my solve procedure is an (Y x Z) matrix - which in the specific case of my data is (D x K). If you're familiar with linear regression, this should seem right!

And to actually use it:

{% highlight scala %}
val z = solve(x, y, 1e2)

val predictions = x * z

//Meanwhile, this won't compile:
val z2 = solve(x.t, y, 1e2)
{% endhighlight %}

And that's it. I can be sure that z has the right shape, because the compiler tells me so, and I can be sure that if I had screwed up the dimensions somewhere, I'll be told at *compile* time, rather than 30 minutes in to a 2-hour, 100 node job on a cluster.

 
###Conclusion

In this post, I've described a problem which, I think, plagues a lot of people who do numerically intensive computing, and proposed a simple solution that relies on the type system to help cope with this problem. Of course, this isn't going to solve all problems - for example, if the solution to some problem is square, and you forget to transpose it, the compiler can't catch that for you.

I haven't yet built this idea into a real system, but I'd be interested in hearing if this idea has already been implemented in scientific or mathematical computing systems, or if not, why people think this is a bad idea. 

Find [me](http://twitter.com/evanrsparks/) on Twitter and make fun of me if you have comments!