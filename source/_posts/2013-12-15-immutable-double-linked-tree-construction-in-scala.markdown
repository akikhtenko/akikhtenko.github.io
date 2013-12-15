---
layout: post
title: "Immutable double-linked tree construction in Scala"
date: 2013-12-15 17:39:55 +0000
comments: true
categories: 
---


Majority of people in the software development community seem to be convinced by now that immutability is good and we need to adhere to immutable data structures even in the languages (Scala among them) where mutability concern is left to developers' discretion. 

However construction of such data structures from some sort of data sources can sometimes be quite tricky. Take for example a double-linked rose tree (multi-way tree). Each note of the tree has an optional reference to the parent node (root of the tree does not have the parent) and a list of children.

In Scala this can be represented as:

```scala
case class Node[A](
	val nodeId: Int, 
	val nodeContent: A, 
	val parent: Option[Node[A]], 
	val children: List[Node[A]])
```

Imagine given is a tree data source in form of the following map of node identifiers to corresponding them children identifiers:

```scala
	val TreeSource =
	  Map(1 -> List(2, 3),
					2 -> List(4, 5, 6),
						4 -> List(),
						5 -> List(7),
							7 -> List(),
						6 -> List(),
					3 -> List())
```

Now how would one inflate an instance of our tree from this map while preserving immutability? Well, this clearly looks like a chicken and egg problem. In order to create a parent node we need to have the list of children already available. The same holds in the opposite direction - all children have to have the parent node instance at the creation time.

Lets try to imagine how we want the function that creates nodes on each level of the tree to look like:

```scala
	def createNodeFor(nodeId: Int, parentOption: Option[Node[String]]): Node[String] = {
		val node: Node[String] = Node(nodeId, s"content ${nodeId}", parentOption, childrenOf(node))
		node
	}
	
	def childrenOf(node: Node[String]): List[Node[String]] = Nil
```

where we try to use the reference `node` recursively in the expression that creates it. The children list is given a `Nil` value which is just a synonym for an empty list. Unfortunately this fails during compilation with an error message:

> forward reference extends over definition of value node

Something tells me that Scala laziness can come in handy here. And indeed if we stick `lazy` keyword in the `node` value declaration the code compiles nicely.

```scala
	def createNodeFor(nodeId: Int, parentOption: Option[Node[String]]): Node[String] = {
		lazy val node: Node[String] = Node(nodeId, s"content ${nodeId}", parentOption, childrenOf(node))
		node
	}
```

However if we try to create such one level tree the code throws at runtime:

```scala
	createNodeFor(1, None)                    //> java.lang.StackOverflowError
                                                  //| 	at scala.collection.Parallelizable$class.$init$(Parallelizable.scala:20)
                                                  //| 
                                                  //| 	at scala.collection.AbstractTraversable.<init>(Traversable.scala:105)
                                                  //| 	at scala.collection.AbstractIterable.<init>(Iterable.scala:54)
                                                  //| 	at scala.collection.AbstractSeq.<init>(Seq.scala:40)
                                                  //| 	at scala.collection.mutable.AbstractSeq.<init>(Seq.scala:47)
                                                  //| 	at scala.collection.mutable.WrappedArray.<init>(WrappedArray.scala:35)
```

The thing is that Scala by default uses call-by-value model where each parameter eagerly evaluates to a value before passing in to a function. This strategy triggers evaluation of the lazy `node` value when `childrenOf()` is invoked and the recursive node creation blows up the stack. Lets switch to the call-by-name strategy and implement `childrenOf()`:

```scala
	def createNodeFor(nodeId: Int, parentOption: Option[Node[String]]): Node[String] = {
		lazy val node: Node[String] = Node(nodeId, s"content ${nodeId}", parentOption, childrenOf(node))
		node
	}
	
	def childrenOf(node: => Node[String]): List[Node[String]] = TreeSource(node.nodeId).map(createNodeFor(_, Some(node)))
```

This code still results in `java.lang.StackOverflowError` but this time it's caused by directly referencing the `node` argument inside `childrenOf()` which also forces eager `node` evaluation. It is obvious that we need to propagate laziness all the way to the `Node` class constructor making necessary changes along the way:

```scala
case class Node[A](
	val nodeId: Int, 
	val nodeContent: A, 
	val parent: Option[() => Node[A]], 
	val children: List[Node[A]])

def createNodeFor(nodeId: Int, parentOption: Option[() => Node[String]]): Node[String] = {
	lazy val node: Node[String] = Node(nodeId, s"content ${nodeId}", parentOption, childrenOf(nodeId, node))
	node
}
	
def childrenOf(nodeId: Int, node: => Node[String]): List[Node[String]] = TreeSource(nodeId).map(createNodeFor(_, Some(() => node)))
```

What we did was we passed `nodeId` explicitly to `childrenOf()` and replaced all `node` referencing to `() => node` to prevent eager evaluation and fixed the types accordingly.

Let's see where we are now:

```scala
val treeRoot = createNodeFor(1, None)     //> treeRoot  : tree.Node[String] = Node(1,content 1,None,List(Node(2,content 2,
                                                  //| Some(<function0>),List(Node(4,content 4,Some(<function0>),List()), Node(5,co
                                                  //| ntent 5,Some(<function0>),List(Node(7,content 7,Some(<function0>),List()))),
                                                  //|  Node(6,content 6,Some(<function0>),List()))), Node(3,content 3,Some(<functi
                                                  //| on0>),List())))
	
assert(treeRoot.children(1).nodeContent == "content 3")
assert(treeRoot.children.head.children(1).parent.get().nodeContent == "content 2")

```

Finally we arrived at the expected result and now can build a tree traversable in both directions.

To be honest I had a different solution where children get populated from inside the `Node` case class and the population function is passed as a val when I started writing this post. But this solution is simpler and cleaner without a doubt.