---
id: 45
title: 'Write Yourself a Scheme in 48 Hours in F# – Part IV'
date: 2011-07-22T07:52:00+00:00
author: lucabol
layout: post
guid: https://lucabolognese.wordpress.com/2011/07/23/write-yourself-a-scheme-in-48-hours-part-iv/
categories:
  - fsharp
tags:
  - fsharp
  - Lambda expressions
  - Parsing
---
It is the evaluator turn. It is a big file, let’s see if I can fit it in a single post.

Aptly enough, the most important function is called _eval_.

<pre class="code">eval env = <span style="color:blue;">function
</span>| String _ <span style="color:blue;">as </span>v <span style="color:blue;">-&gt; </span>v
| Number _ <span style="color:blue;">as </span>v <span style="color:blue;">-&gt; </span>v
| Bool _ <span style="color:blue;">as </span>v <span style="color:blue;">-&gt; </span>v
| Atom var <span style="color:blue;">-&gt; </span>getVar var env
| List [Atom <span style="color:maroon;">"quote"</span>; v] <span style="color:blue;">-&gt; </span>v
| List [Atom <span style="color:maroon;">"if"</span>; pred; conseq; alt] <span style="color:blue;">-&gt; </span>evalIf env pred conseq alt
| List [Atom <span style="color:maroon;">"load"</span>; fileName] <span style="color:blue;">-&gt; </span>load [fileName] |&gt; List.map (eval env) |&gt; last
| List [Atom <span style="color:maroon;">"set!" </span>; Atom var ; form] <span style="color:blue;">-&gt; </span>env |&gt; setVar var (eval env form)
| List [Atom <span style="color:maroon;">"define"</span>; Atom var; form] <span style="color:blue;">-&gt; </span>define env var (eval env form)
| List (Atom <span style="color:maroon;">"define" </span>:: (List (Atom var :: parms) :: body)) <span style="color:blue;">-&gt;
    </span>makeNormalFunc env parms body |&gt; define env var
| List (Atom <span style="color:maroon;">"define" </span>:: (DottedList ((Atom var :: parms), varargs) :: body)) <span style="color:blue;">-&gt;
    </span>makeVarargs varargs env parms body |&gt; define env var
| List (Atom <span style="color:maroon;">"lambda" </span>:: (List parms :: body)) <span style="color:blue;">-&gt; </span>makeNormalFunc env parms body
| List (Atom <span style="color:maroon;">"lambda" </span>:: (DottedList(parms, varargs) :: body)) <span style="color:blue;">-&gt; </span>makeVarargs varargs env parms body
| List (Atom <span style="color:maroon;">"lambda" </span>:: ((Atom _) <span style="color:blue;">as </span>varargs :: body)) <span style="color:blue;">-&gt; </span>makeVarargs varargs env [] body
| List (func :: args) <span style="color:blue;">-&gt;
    let </span>f = eval env func
    <span style="color:blue;">let </span>argVals = List.map (eval env) args
    apply f argVals
| badForm <span style="color:blue;">-&gt; </span>throw (BadSpecialForm(<span style="color:maroon;">"Unrecognized special form"</span>, badForm))</pre>

This is the core of the evaluator. It takes as an input the LispVal generated by the parser and an environment and returns a LispVal that is the result of the reduction. As a side effect, it occasionally modify the environment. I carefully crafted the previous phrase to maximize the discomfort&#160; of the functional programmers tuned in. Such fun 🙂

More seriously (?), here is what it does:

  * If it is a String, Number of Bool, just return it 
  * If it is an Atom, return its value 
  * If it is a _quote_&#160; statement, return what is quoted (read your little schemer manual) 
  * If it is an _if_ statement, evaluate it (see below) 
  * If it is a _load_ statement, load the file (see below) and evaluate each expression using the current environment. Return the last expression in the file 
  * If it is a _set!_, set the value of the passed variable to the evaluated form 
  * If it is a _define_, do almost the same as above (except that you don’t throw if the variable doesn’t exist, but you create it) 
  * If it is a _define_&#160; that defines a function (it has that shape), create a ‘function slot’ in the environment. That is a (functionName,&#160; FuncRecord) pair (see below) 
  * If it is a _lambda_, return the FuncRecord that describe the inline function 
  * If it is a function call, evaluate the expression that describe the function (yes, you can do that in Lisp!!), evaluate the arguments, and apply the function to the arguments 
  * Otherwise, it must be a bad form, throw it back to the calling function to do something meaningful with it 

We have a bunch of ‘see below’ to take care of. We’ll look at them in order.

First the ‘if’ statement. If the evaluated predicate is Bool(True) evaluate the consequent, otherwise evaluate the alternative. For some reason, I wrote it the other way around.

<pre class="code"><span style="color:blue;">and
    </span><span style="color:green;">// 1a. If the evaluation of the pred is false evaluate alt, else evaluate cons
    </span>evalIf env pred conseq alt =
        <span style="color:blue;">match </span>eval env pred <span style="color:blue;">with
        </span>| Bool(<span style="color:blue;">false</span>) <span style="color:blue;">-&gt; </span>eval env alt
        | _ <span style="color:blue;">-&gt; </span>eval env conseq</pre>

Then there is the _load_ function. It reads all the test and gets out the list of LispVal contained in it.

<pre class="code"><span style="color:blue;">let </span>load = fileIOFunction (<span style="color:blue;">fun </span>fileName <span style="color:blue;">-&gt; </span>File.ReadAllText(fileName)
                                           |&gt; readExprList)</pre>

_ReadExprList_ is part of the parser. We’ll look at it then. Sufficient here to say that it takes a _string_ and returns a list of _LispVal_.

_FileIOFunction_ just encapsulates a common pattern in all the file access functions. I don’t like such mechanical factorization of methods, without any real reusability outside the immediate surroundings of the code. But I like repetition even less.

<pre class="code"><span style="color:blue;">let </span>fileIOFunction func = <span style="color:blue;">function
    </span>| [String fileName] <span style="color:blue;">-&gt; </span>func (fileName)
    | [] <span style="color:blue;">-&gt; </span>throw (IOError(<span style="color:maroon;">"No file name"</span>))
    | args <span style="color:blue;">-&gt; </span>throw (NumArgs(1, args))</pre>

A family of functions create FuncRecord given appropriate parameters. I seem to have lost memory of the contortions related to the last one. If I end up having to work again on this code, I’ll need to figure it out again. Note to myself, please comment this kind of code next time.

<pre class="code"><span style="color:blue;">let </span>makeFunc varargs env parms body =
            Func ({parms = (List.map showVal parms); varargs = varargs; body = body; closure = env})
<span style="color:blue;">let </span>makeNormalFunc = makeFunc None
<span style="color:blue;">let </span>makeVarargs = showVal &gt;&gt; Some &gt;&gt; makeFunc</pre>

_apply_ is the other workhorse function in the evaluator.&#160; The best way to understand it is to start from the bottom (where _bindVars_ starts the line). We are binding the arguments and the variable arguments in the closure that has been passed in. We then evaluate the body. But the body is just a list of _LispVal_, so we just need to evaluate them in sequence and return the result of the last one.

<pre class="code"><span style="color:blue;">and </span>apply func args =
    <span style="color:blue;">match </span>func <span style="color:blue;">with
    </span>| PrimitiveFunc(f) <span style="color:blue;">-&gt; </span>f args
    | Func ({parms = parms; varargs = varargs; body = body; closure = closure}) <span style="color:blue;">-&gt;
        let </span>invalidNonVarargs = args.Length &lt;&gt; parms.Length && varargs.IsNone
        <span style="color:blue;">let </span>invalidVarargs = args.Length &lt; parms.Length && varargs.IsSome
        <span style="color:blue;">if </span>invalidVarargs || invalidNonVarargs
        <span style="color:blue;">then
            </span>throw (NumArgs(parms.Length, args))
        <span style="color:blue;">else
            let </span>remainingArgs = args |&gt; Seq.skip parms.Length |&gt; Seq.toList
            <span style="color:blue;">let </span>evalBody env = body |&gt; List.map (eval env) |&gt; last
            <span style="color:blue;">let rec </span>zip xs1 xs2 acc =
                <span style="color:blue;">match </span>xs1, xs2 <span style="color:blue;">with
                </span>| x1::xs1, x2::xs2 <span style="color:blue;">-&gt; </span>zip xs1 xs2 ((x1, x2)::acc)
                | _ <span style="color:blue;">-&gt; </span>acc
            <span style="color:blue;">let </span>bindVarArgs arg env =
                <span style="color:blue;">match </span>arg <span style="color:blue;">with
                </span>| Some(argName) <span style="color:blue;">-&gt; </span>bindVars [argName, (List remainingArgs)] env
                | None <span style="color:blue;">-&gt; </span>env
            bindVars (zip parms args []) closure
                |&gt; bindVarArgs varargs
                |&gt; evalBody
    | funcName <span style="color:blue;">-&gt; </span>throw (NotFunction(<span style="color:maroon;">"Expecting a function, getting "</span>, showVal funcName))</pre>

This is enough for one post. Next time we’ll finish the evaluator.