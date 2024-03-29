---
layout: single
title: 符号和解释
date: 2022-03-06 22:52:07
categories: "Caprice"
tags:
- CS
- PL
- Research
header:
  teaser: https://ilyasergey.net/pnp/coq-logo.png
---

在被[Coq](https://coq.inria.fr/)折磨了快一个月后，终于我终于看完了Logic Foundation。作为[Software Foundations](https://softwarefoundations.cis.upenn.edu/)系列书籍中的第一员大将，它还是有点东西的。这里只浅谈一下我对于符号和解释这两个概念所产生的更深的理解。正文部分基本不涉及任何与编程本身相关的问题。

# Preface

我并不打算对书的中知识概括总结，而是希望浅谈自己在读书过程中对于两个重要的概念，符号与解释的一些理解和思考。

在这之前先简介一下Coq：简单的说它是一个functional programming language。但是它奇怪的地方在于它也可以被当成一个辅助证明工具用。由于一些项目需要我用到Coq（当然我也对它比较感兴趣），于是开始学习[Coq in a Hurry](https://cel.archives-ouvertes.fr/inria-00001173v6/document)这个文档。然而并没有什么卵用：它并不能帮我learn Coq in a hurry。于是我便走向了Software Foundations的~~康~~羊~~庄~~肠~~大~~小道。

刚开始看到这玩意的时候我直接乐呵了：软件基础？这是有多么基础？第一本书还就叫逻辑基础（Logic Foundations）。笑死，不就是一本基础书籍么，这不是分分钟学穿了？后来我逐渐理解一切：在PL这个领域，所有宣称自己是基础的都是骗子。这门课在宾大的课号是CS500：这意味着它是研究生阶段和顶级本科生的课程。当然，等我被骗上贼船发现这一切的时候已经晚了。

Anyway，这仍旧是一本非常不错的书，因为它十分阳间。似乎它在每次开课前都能得到更新，这也就意味着书中的内容在最新的stable Coq上仍然有效。不像某些阴间教材一样，即使是2021年出版的书，还非得用着自己写的，在N年前已经停止支持软件（还只支持windows）。这里我说的就是Thinking Programs！书虽然是挺不错的，但是里面的Code实在是过于陈旧了，用的都是祖传软件上的入土代码，对Commandline和Mac而言极不友好。

感谢SF一书的作者们为其提供public access，让人类能自由免费地学习其中的知识。

# 符号

在幼儿时期的我经常思考一个问题：“我“究竟是什么？这里并不牵扯到哲学上的任何意义，将这个问句中的”我”替换成其他任何词语并不会影响这个问题的本质，比如“红色”究竟是什么？或“dASYg”究竟是什么？

聪明的小朋友可能已经发现了，这里用引号圈出来的【东西】，就单纯的是一个符号，它或许不具备任何的意义。不过这个问题就十分奇怪，为什么”我“这个符号能够表达【我自己】的意义？是因为谁先为”我“赋予了【我】的意义呢？所有说汉语的人都明白”我“代表【我】吗？大大的问题对我幼小心灵产生了极大的冲击，直到我知道还有一种叫做色盲的特殊人类时，才开始渐渐明白这个问题。

在他们（more specifically，红绿色盲）眼里，“红色”并不意味着我眼里的【红色】。而对于那些不认识汉字的人而言，”红色“甚至都不能表达【红色】的意义。它们充其量只是符号————甚至他们无法知道这究竟是一个符号还是俩符号。实际上，符号或许在不同人眼里就是不同的。而这里的所有文字，也不过只是堆彻而成的一堆符号，我甚至很庆幸在观察着这一堆符号的”你“能够猜测到我想表达的意思。

## Formally

先放一个wiki上关于formal symbol的[链接](https://en.wikipedia.org/wiki/Symbol_(formal))。这里formal被译作形式化，我曾经一直不太理解这个词语真正的含义，但现在逐渐能够理解。形式化的符号就是脱离了意义的符号，它们只和这个符号长啥样有关，基本上可以视作【得意忘形】这个词语的反面。

而在寻求形式化的过程中，很多教材中会提到语言。这里也是最有意思的地方：语言学，计算机，数学，甚至于哲学居然在这个奇奇怪怪的地方交汇了！形式化的符号如果想要组成一个句子（将符号组合成为更加复杂的符号），那么就必须依赖于语法（[formal grammar](https://en.wikipedia.org/wiki/Formal_grammar)）。这里的语法和我们年轻时学习外语中学到的语法并无二致：即是一套符号的重写规则。例如在英文中，最简单的“主谓宾”语句，其主、谓、宾三个字（符号）都可以被符合要求的另一个符号重写，从而形成另一个具体的句子。

另一方面，数理逻辑也十分依赖于符号。逻辑的推导一方面可以看作是语义（意义）上的关系，另一方面就是单纯基于一些规则的符号演算（Sequent calculus)。而这些东西再往深了去扯一扯的话，就涉及哲学了。有一个分支叫做【[形而上学](https://zh.wikipedia.org/wiki/%E5%BD%A2%E4%B8%8A%E5%AD%B8)】，和符号/逻辑有着不小的关系。鉴于我并不懂这玩意，就不在这里瞎说了。
但是我认为这个词语是有歧义的。如何去理解“而上”呢？是理解为【建立在XXX之上】，还是说【在XXX之上的另一个境界】？英文是Metaphysics，我更倾向于后者的意思。

在此，我想用“1 + 1 = 2”这个例子收本节的尾。从小学二年级我就不明白这个等式为什么成立。然而在思考为什么一加一等于二之前，或许更值得关注的问题在于，为什么“1”代表【一】。

# 解释

简单的说，解释就是在为符号赋予意义（[wiki](https://zh.wikipedia.org/wiki/%E8%A7%A3%E9%87%8B_(%E9%82%8F%E8%BC%AF))）。或许当我在问为什么“XXX”代表【XXX】这个意思时，一个最合理的答案便是：我们将【XXX】这个意思赋予了“XXX”这个符号本身。回到上面那个例子，“1”表示【一】，“2”表示【二】本质上都是人为了方便而将一些意义用符号表示。而同一个意义可能被不同的符号表示，同一个符号也可以在不同的解释下表示不同的含义。例如罗马数字和阿拉伯数字的符号不同，但数值相等的数意义却一样；“艹”这个符号就在中文和日文中（笑）表示不同的意义，即它有着两种不同的解释。值得一提的是，刚刚括号内的（“笑”）并非想表达我真的笑了，而是想表示“艹”在日文中可以表示【笑】的意思。由此可见歧义真的是无处不在。

> 人类是无法互相理解的。

在形式化的逻辑过程中，解释并非是必要的，然而人类或许需要某种解释。对于这句名言我有了更深刻的体会。如何知道别人用符号表达的“东西”就和ta脑子里面对于这些符号的解释是同一个“东西”？语言在许多时候都显得有那么一些苍白。首先，其并不一定能精确地表达一个人的真实想法，也无法保证会被另一个接受信息的人按照原本语言想要被表达的意思所理解。在想到这里时，我顿悟了三体人物种的优越性：他们可以感受到同类的脑中的想法，这种脱离了语言的意识表达能够更加精确地传达意义；也明白了巴别塔为什么不能被建立：语言都不通的话，相互理解更是无稽之谈。

## 愚蠢的机器

在很久之前接触到“图形计算器”这个单词的时候，还觉得它是一个十分酷炫的概念。其中有一个功能是“符号计算”。这个词语对彼时还是人类幼崽我的心灵造成了更大的冲击：啥是符号计算？我久久不能理解它，并需要一个【解释】。难道加减乘除不是符号？这里的符号是啥玩意呢？后来我才逐渐理解一切，简单的说它与数值计算相对应，可以在计算中掺入符号（e.g., X， Y， Z）。掺入了符号的多项式化简就需要使用这种能力。

> 电脑很笨的，只认识0和1。

这是在年轻时某人对我说过的话。起初我并不能理解这句话，因为电脑上能显示出各种语言的文字和段落，它能做各种计算，甚至“理解”我需要做什么。不过现在想想或许这句话也有道理，这些文本和计算并非被机器所理解了，而是作为一个个符号被展示（print）出来而已。真正解释这些符号背后的意义的并不是计算机，而是人类，而它们所真正【理解】的东西就只有0和1了。不过话又说回来，既然人类无法相互理解，人类就能理解计算机吗？人类为什么就能知道计算机无法理解这些符号的意义呢？或许只有真正设计出芯片的人们更有发言权吧，而我们只知道这个黑箱子会产生可以被我们所【理解】的内容。

在计算机中，神经网络的局限性多被认为是【不可解释】。虽然有一些解释它们的方法被开发出来，似乎这仍然是困扰学者的一个大问题。人们既然无法理解某一坨模型到底在想什么，那么凭什么信任它能够产生足够正确的结果呢？我们在喂给了机器大量的数据（符号）之后，机器也能够习得这些符号之间的推演规则。然而我们并不能理解机器为什么会输出某些奇奇怪怪的符号，甚至于机器本身也不能理解这些符号的意义。所以，机器或许也需要一个解释吧。待到机器真正能够【解释】的那天，或许【计算机】就不只是“计算机”了。

# 后记

话又说回来，我们为什么认为自己【理解】了符号呢？有没有可能，只是人类的大脑把这些符号替换成了更加复杂的符号（low level representation），而我们只是根据特定的规则作出处理这些符号之后的动作？（笑）
