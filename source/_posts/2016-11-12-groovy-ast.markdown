---
layout: post
title: "[译]使用Groovy的AST Transformation做DSL操作"
date: 2016-11-12 21:21:34 +0800
comments: true
categories: java
---

最近在看一些java开源项目时，发现不少是用gradle做为项目构建工具的。之前虽然也用过gradle，但是却没怎么仔细留意build.gradle的语法是groovy的。但这次再怎么看也觉得里面的好多语法都和以前用过的groovy都联系不到一起。各种懵逼状态。。。后来阅读了这篇文章，算是解答了一些疑问：<http://www.cnblogs.com/CloudTeng/p/3418072.html>。但是对于下面这种写法，还是不知道是怎么回事：

    task copyFile(type: Copy){
        from 'xml'
        into 'destination'
    }

copyFile做为task名称竟然不是一个字符串，阅读了groovy的文档也没发现字符串可以省略引号的说明(php中引号倒是可以省略），此外一个方法后面跟一个参数然后这个参数又跟着一个括号，这又是什么语法。。。凭直觉觉得这里的copyFile应该是一个方法，但是这时候copyFile还没有定义啊。。。

带着以上疑问，去翻了一下groovy的官方文档，凭感觉觉得gradle是利用了groovy的ast trasnfomation，也就是抽象语法树转换(故名思议，也就是能够转换groovy的语法树从而创造自己的一套语法)。那么到底是不是这样呢？<http://blog.csdn.net/zxhoo/article/details/29830529>给出了解释并证明了这个结论。但是groovy的ast transformation到底是什么东西呢？国外有一篇博客给出了比较清晰明了的讲述：<http://www.tuicool.com/articles/VN7Vvq>。此文即对此篇博文的翻译。

<!--more-->

-------------

为了给出此问题的一个上下文，在我目前的项目上创建了一个使用Groovy作为主要语言的内嵌DSL。这个DSL和MongoDB lingo类似，下面是一个例子：

    // Sample Delta file 
    use test
    db.customers.add("{'city' : 'Please Set City', 'pin':  'Please Pin code' }")

    //Increment age field by 1
    db.customers.transform('age', "{ $add: ["$age", 1] }")

    // Set context to transactions db
    use transactions

    // add to orders collection a nested document
    db.orders.add('{"dispatch" : { "status" : "Default", "address": { "line1" : "Road", "city": "City" }}}')

和Mongo Shell类似的，我想要支持在命令参数中使用单引号和双引号包裹住的字符串。和javascript一样，你可以在字符串内部使用引号，只要不要和外部包裹字符串的引号匹配就可以。为了实现这些，我现在遇到两个问题：

1. ***use*** 是Groovy的一个供Groovy Categories使用的默认方法，和Scala中的implicit以及c#中的扩展方法类似。
2. 在add、tranform函数中的双引号参数是Groovy中的GString，可以使用$来做字符串替换-在Groovy的世界中，你可能听过"You need a $ in GString ;)"这种说法。它会解析出现在$后面的表达式然后替换为表达式的字符串输出。此外，GString是延迟解析的，只有当toString被调用或者做为参数传递给函数的时候，GString才会对其中的$做解析。因此，上面的例子中age并没有定义，会在GString被解析的时候产生问题。

当然，我们可以做一些hack的事情来解决上面的问题。我们不用use而是换成using来解决第一个问题。但是第二个问题，我怎样才能阻止人们不在函数参数中使用双引号字符串呢？在文档中注明规范意味着被动并且依赖于遵守规范的开发者。因此，这样做并不很hack。上面两个问题看起来都像是编译级别的问题。下面就讲述我是如何一石二鸟解决这些问题的。

Groovy提供了访问抽象语法树并转换它的方法。一个AST是编译器在编译阶段生成的中间表示。这里讲的AST指的是能够产生另外的翻译或者字节码。Groovy以[ASTTransformation](http://groovy.codehaus.org/gapi/org/codehaus/groovy/transform/ASTTransformation.html)的形式提供了一个钩子让我们可以在编译阶段添加、修改语法树。一个实现了此接口的类必须以[@GroovyASTTransformation](http://groovy.codehaus.org/gapi/org/codehaus/groovy/transform/GroovyASTTransformation.html)注解，这样Groovy才能知道应该在哪一个阶段运行。这样我可以处理全局AST转换，其中visit方法会为sourceUnit(原始的源代码)调用一次，并且我会忽略ASTNode[]中的第一个和第二个元素。下面是我的ASTTransformation代码：

    @Slf4j
    @GroovyASTTransformation
    public class StatementTransformation implements ASTTransformation {
      private def transformations = ['use' : 'using']

      @Override
      void visit(ASTNode[] nodes, SourceUnit source) {
        log.info("Source name = ${source.name}")
        ModuleNode ast = source.ast
        def blockStatement = ast.statementBlock

        blockStatement.visit(new CodeVisitorSupport() {
          void visitConstantExpression(ConstantExpression ce) {
            def name = ce.value
            if (transformations.containsKey(name)) {
              def newName = transformations[name]
              log.debug("Transform Name => $name -> $newName")
              ce.value = newName
            } else {
              log.debug("Skip Name => $name")
            }
          }

          public void visitArgumentlistExpression(ArgumentListExpression ale) {
            log.debug("Arg List $ale.expressions")
            def expressions = ale.expressions
            expressions.eachWithIndex { expr, idx ->
              if(expr.getClass() == GStringExpression) {
                log.debug("Transform GString => String ($expr.text)")
                expressions[idx] = new ConstantExpression(expr.text)
              }
            }
            log.debug("Transformed Arg List $ale.expressions")
            super.visitArgumentlistExpression(ale)
          }
        })
      }
    }


1. 当遇到like, use, db, customers, add, transform, fn params等常量时，visitConstantExpression(...)会被调用。根据已经定义的transformations map(第四行)，相应的值会被简单重新赋值。(18行)
2. 当调用函数时，visitArgumentlistExpression会被调用。在我的例子中db.customers.transform(...)和db.customers.add(...)是函数调用并且整个所有的参数都被传给了visitArgumentlistExpression方法。在GStringExpression出现的时候将它转换为了ConstantExpression(30行)。

接下来的代码是如何使用上面的代码做转换。

Reader读取所有的DSL文件，在的例子中，我们把它们叫做delta文件。对于每一个deleta文件，我创建了一个新的GroovyShell并让它去解析代码(delta文件中的)。这里的shell用我自定义的AST transformer做了相应的配置。shell解析出一个对象并传递给Parser。这样Pardser得到的结点，其中的GString已经全被转换为了普通String，'use'也已经被转换为了'using'方法。

    @Slf4j
    public class Reader {
      private def createNewShell() {
        def secureCustomizer = new SecureASTCustomizer()
        secureCustomizer.with {
          methodDefinitionAllowed = false // user will not be able to define methods
          importsWhitelist = [] // empty whitelist means imports are disallowed
          staticImportsWhitelist = [] // same for static imports
          staticStarImportsWhitelist = []
          ....
        }

        def astCustomizer = 
          new ASTTransformationCustomizer(new StatementTransformation())
        def config = new CompilerConfiguration()
        config.addCompilationCustomizers(secureCustomizer, 
                              astCustomizer)
        new GroovyShell(config)
      }

      public Tree read(final List<File> deltas) {
        def parser = new Parser()
        deltas.each { delta ->
          def deltaName = delta.name
          def dslCode = """{-> $delta.text}"""
          //shell evaluates once, hence create new each time
          def shell = createNewShell()
          def deltaObject = shell.evaluate(dslCode, deltaName)
          try {
            parser.parse(deltaObject)
          } catch (Throwable t) {
            throw new InvalidGrammar("$deltaName --> ${t.message}")
          }
          shell = null
        }
        parser.ast()
      }
    }

下面的是Parser的代码。In here is the using(db) method that gets called after the custom transformation is applied. An astute reader may have noticed how I intercept property access using the getProperty method (a part of the the Groovy MOP - Meta-Object Protocol feature) to change the database context.这里在自定义转换之后调用using(db)。聪明的读者会发现我是如何使用getProperty(Groovy元对象协议编程的一部分，和invokeMethod、methodmissing类似)来拦截住对象方法的访问来改变数据库上下文的。

    @Slf4j
    class Parser {
      private Tree tree = new Tree()
      private def dbContext

      @CompileStatic
      def getProperty(String name) {
        log.debug("property name is: $name")
        if(name == 'db') {
          return dbContext
        }
        tree.using(name)
      }

      def using(db) {
         log.info "Setting db context to ${db.toString()}"
         dbContext = db
      }

      public Tree parse(Closure closure) {
        def cloned = closure.clone()
        cloned.delegate = this
        cloned.resolveStrategy = Closure.DELEGATE_FIRST
        cloned()
        tree
      }

      def ast() {
        tree
      }
    }
