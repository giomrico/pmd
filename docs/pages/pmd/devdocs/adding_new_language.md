---
title: PMD Adding a New Language
short_title: Adding a New Language
tags: [customizing]
summary: Adding a New Language to PMD
last_updated: July 3, 2016
sidebar: pmd_sidebar
permalink: pmd_devdocs_adding_new_language.html
folder: pmd/devdocs
---

# How to Add a New Language to PMD

## 1.  Start with a new sub-module.
*    See pmd-java or pmd-vm for examples.

## 2.  Implement an AST parser for your language
*   Ideally an AST parser should be implemented as a JJT file *(see VmParser.jjt or Java.jjt for example)*
*   There is nothing preventing any other parser implementation, as long as you have some way to convert an input stream into an AST tree. Doing it as a JJT simplifies maintenance down the road.
*   See this link for reference: [https://javacc.java.net/doc/JJTree.html](https://javacc.java.net/doc/JJTree.html)

## 3.  Create AST node classes
*   For each AST node that your parser can generate, there should be a class
*   The name of the AST class should be “AST” + “whatever is the name of the node in JJT file”.
    *   For example, if JJT contains a node called “IfStatement”, there should be a class called “ASTIfStatement”
*   Each AST class should have two constructors: one that takes an int id; and one that takes an instance of the parser, and an int id
*   It’s a good idea to create a parent AST class for all AST classes of the language. This simplies rule creation later. *(see SimpleNode for Velocity and AbstractJavaNode for Java for example)*
*   Note: These AST node classes are generated usually once by javacc/jjtree and can then be modified as needed.

## 4.  Compile your parser (if using JJT)
*   An ant script is being used to compile jjt files into classes. This is in alljavacc.xml file.
*   In the file, create a new target for your language. Use vmjjtree or javajjtree as an example.
*   Inside the alljavacctarget, add your new target to the “depends” list just before cleanup

## 5.  Create a TokenManager
*   Create a new class that implements the `TokenManager` interface *(see VmTokenManager or JavaTokenManager for example)*

## 6.  Create a PMD parser “adapter”
*   Create a new class that extends AbstractParser
*   There are two important methods to implement
    *   `createTokenManager` method should return a new instance of a token manager for your language *(see step #5)*
    *   `parse` method should return the root node of the AST tree obtained by parsing the Reader source
    *   See `VmParser` class as an example

## 7.  Create a rule violation factory
*   Extend `AbstractRuleViolationFactory` *(see VmRuleViolationFactory for example)*
*   The purpose of this class is to createa rule violation instance specific to your language

## 8.  Create a version handler
*   Extend `AbstractLanguageVersionHandler` *(see VmHandler for example)*
*   This class is sort of a gateway between PMD and all parsing logic specific to your language. It has 3 purposes:
    *   `getRuleViolationFactory` method returns an instance of your rule violation factory *(see step #7)*
    *   `getParser` returns an instance of your parser adapter *(see step #6)*
    *   `getDumpFacade` returns a `VisitorStarter` that allows to dump a text representation of the AST into a writer *(likely for debugging purposes)*

## 9.  Create a parser visitor adapter
*   If you use JJT to generate your parser, it should also generate an interface for a parser visitor *(see VmParserVisitor for example)*
*   Create a class that implements this auto-generated interface *(see VmParserVisitorAdapter for example)*
*   The purpose of this class is to serve as a pass-through `visitor` implementation, which, for all AST types in your language, just executes visit on the base AST type

## 10. Create a rule chain visitor
*   Extend `AbstractRuleChainVisitor` *(see VmRuleChainVisitor for example)*
*   This class should `implement` two `important` methods:
    *   `indexNodes` generates a map of "node type" to "list of nodes of that type". This is used to visit all applicable nodes when a rule is applied.
    *   `visit` method should evaluate what kind of rule is being applied, and execute appropriate logic. Usually it will just check if the rule is a "parser visitor" kind of rule specific to your language, then execute the visitor. If it’s an XPath rule, then we just need to execute evaluate on that.

## 11. Make PMD recognize your language
*   Create your own subclass of `net.sourceforge.pmd.lang.BaseLanguageModule`. *(see VmLanguageModule or JavaLanguageModule as an example)*
*   You’ll need to refer the rule chain visitor created in step #10.
*   Add for each version of your language a call to `addVersion` in your language module’s constructor.
*   Create the service registration via the text file `src/main/resources/META-INF/services/net.sourceforge.pmd.lang.Language`. Add your fully qualified class name as a single line into it.

## 12. Create an abstract rule class for the language
*   Extend `AbstractRule` and implement the parser visitor interface for your language *(see AbstractVmRule for example)*
*   All other rules for your language should extend this class. The purpose of this class is to implement visit methods for all AST types to simply delegate to default behavior. This is useful because most rules care only about specific AST nodes, but PMD needs to know what to do with each node - so this just lets you use default behavior for nodes you don’t care about.

## 13. Create rules
*   Rules are ceated by extending the abstract rule class created in step 12 *(see `EmptyForeachStmtRule` for example)*
*   Creating rules is already pretty well documented in PMD - and it’s no different for a new language, except you may have different AST nodes.

## 14. Test the rules
*   See BasicRulesTest for example
*   You have to create a rule set for your language *(see vm/basic.xml for example)*
*   For each rule in this set you want to test, call `addRule` method in setUp of the unit test
    *   This triggers the unit test to read the corresponding XML file with rule test data *(see `EmptyForeachStmtRule.xml` for example)*
    *   This test XML file contains sample pieces of code which should trigger a specified number of violations of this rule. The unit test will execute the rule on this piece of code, and verify that the number of violations matches
*   To verify the validity of the created ruleset, create a subclass of `AbstractRuleSetFactoryTest` (*see `RuleSetFactoryTest` in pmd-vm for example)*.
    This will load all rulesets and verify, that all required attributes are provided.

    *Note:* You'll need to add your ruleset to `rulesets.properties`, so that it can be found.
