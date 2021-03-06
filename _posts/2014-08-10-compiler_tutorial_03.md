---
layout: post
title: " 基于LLVM开发编译器（03）：词法分析器的实现"
description: "LLVM Compiler Tutorial 03"
category: compiler
tags: [compiler,tutorial,llvm]
---
{% include JB/setup %}
经过第二节的铺垫，我们终于迎来了第一个编译器部分的实现：词法分析器。该部分的代码已经上传到我的GitHub上，可以点击这个[链接][1]查看。

如教程的第一篇文章所述，我们将会以Pascal原型为基础，进行一部分的裁剪与自定义扩充，裁剪与扩充的原则也是以编译原理的需要为主，而该编译器的主要参考Pascal标准主要是以下这两个链接：[Pascal标准][2]和[GNU Pascal][3]，而我也不会在博客这里讲述Pascal的基础知识，我认为能阅读我此教程，并且一起参考编译器实现的同学应该不需要我苦口婆心的来讲述什么是Pascal的变量，什么是Pascal的循环之类的了。而我这里也提供了两个链接，方便有需要的同学查阅，请点此[链接][4]和这个教程[链接][5]。而我为什么会选择Pascal呢？我也其实考虑了很久，我觉得最直接的一个理由就是Pascal简单，并且具有严格的结构化特征，并且Pascal具有着“无脑化”，我们只需要按部就班的实现即可。与此同时，Pascal具有着我想要讲的一切东西，无论是指针类型、引用类型、函数声明Forward、自定义类型Type、数组类型、File、Text等类型乃至我想讲的与C的交互，Pascal都有了。当完成编译器前端部分，代码生成后，我们的重点就开始放在了代码优化上面，放在了LLVM IR，所以无论是Pascal抑或C这样的语言，后面对我们的影响都不太大了，只要我们能生成到LLVM IR。而看我说了这么多，自然整个系列教程是浩瀚的，但是我希望你们能跟着我的脚步，一起感受到编译器的魅力所在！

对于一个词法分析器来说，其作用如第二篇文章所述，其在于读取源文件的字符，然后将这些字符组成一个一个符合特定程序语言的词素，最后生成输出一个有效的词法单元序列。也就是说，对于词法分析器来说，我们需要输出的是一个又一个的词法单元，我们称其为Token。而Token具有着一些属性，如词法单元类型，词法单元值等，每个编译器实现者抑或编译原理教材其实对于Token的属性并未有一致的认识，而在我认知的编译世界中，我认为Token具有的属性是有以下几种：词法单元类型（Token Type），词法单元值（Token Value），词法单元所在源文件位置（Token Location），词法单元符号优先级（Token Symbol Precedence），词法单元名称（Token Name）。首先，词法单元类型包括了Integer，Real（即对应C/C++的double），Identifier等，词法单元值则为词法单元本身在编译器的编码，如读取到if这个关键字，会对应到TokenValue::If，而这个编码对于用户是不可见的，而词法单元名称这个属性则是做显示词法单元值的事情。每个词法单元在源文件也具有着位置信息，其具体表现为词法单元所在的行号与列号。在这儿，我与一些编译器作者的不同观点在于，我认为每个词法单元是具有优先级的，而优先级在很多编译器作者的眼中是放在Parser部分，以独立的符号优先级表给出，而对于我来说，我则认为每个Token与生俱来则是带有着符号优先级，比如+是具有10的优先级，而*则是具有20的优先级，而if这样的关键字，抑或123数值常量，abc标识符则是具有着-1的优先级。这里面我想要说明的是无论哪种方式都没有对错，只是对编译世界的认知哲学不同罢了。

那么既然这样，我们在词法分析器中的实现自然需要以相应的形式来表示这些信息，那么接下来我们则看代码是如何实现的。不过在讲代码实现之前，我想我应该花一小会儿来讲一下我的代码书写风格（主要是命名风格）。因为C++并没有严格规定代码需要是怎么样的书写风格（包括命名），这也造成了C++代码的“自由风”，而这样的洒脱与自由自然不是太好的现象，于是出现了[Google C++ Style Guide][6]这样的规范，而对于我来说，我的编码则主要是遵循下面的特点：1. 对于enum/union/struct/class来说，我的命名是每个单词的头字母都大写，其余字母小写，如class NameTest。2. 对于enum里面的成员，每个字母都大写，单词之间使用下划线分隔。3. 对于union/struct/class的数据成员，遵循Camel命名法，并且在末尾加上下划线。如buffer\_，symbolPrecedence\_。4. 函数、函数参数、变量遵循Camel命名法。5. 代码实现中能不用指针则尽量不用指针。（而在这点儿上，会与LLVM有矛盾，因为LLVM暴露的接口是指针，但是对于与LLVM无关的地方，我会尽量避免指针）。以上则是代码主要遵循的一些准则。

讲完了代码风格，我们可以正式来看看我们如何用代码来表示这些信息了。对于有关词法单元的信息，我都放在了token.h这个文件里面，其实现在token.cpp文件里。在token.h中，我们首先使用TokenType的enum class来表示。enum class是C++11引入的一个新特性，你可以理解为更安全的enum，那么如果你真的不想查询有关enum class的信息，那么就按照enum来理解吧，不会出现太大的障碍。而在TokenType中，则对应了我们词法单元的类型，如整型，浮点型，关键字，操作符等类型，而我们所有的词法单元类型都可以在这里面找到所在。我们举一个类比的例子，这就好比我们的英文单词具有名词、动词、形容词、副词等类型，如case在英文单词中，则是对应着名词类型，然而在我们的程序（编译器）世界，它对应的则是关键字（TokenType::KEYWORDS)类型。这样我们就能清楚，TokenType其实就是表示这些类型的一个集合，英语世界是名词、动词等，而程序世界则变成了关键字、标识符等类型了，而具体的代码很简单，  

{% highlight cpp %}
enum class TokenType
{
   INTEGER,
   REAL,
   ...
   IDENTIFIER,
   KEYWORDS,
   ...
}; 
{% endhighlight %}

而我们表示了词法单元类型，接下来则是表示词法单元值。与词法单元类型类似，也只需要一个enum class即可。  

{% highlight cpp %}
enum class TokenValue
{
   AND,
   FOR,
   ...
   LEFT_PAREN,
   ...
   PLUS,        
   ...
};
{% endhighlight %}

到此为止，我们则完成了词法单元的两个类型表示，分别是词法单元类型，与词法单元值。而对于词法单元来说，这两个是可以匹配的，比如现在有词法单元值TokenValue::AND，则它对应的则是词法单元类型TokenType::KEYWORDS。那么这两个如何怎么联系起来呢？当然有可能有同学立即反应出来是用std::map。没错，这是正确的，但是如上文所示，在我的编译认知哲学中，每个Token不仅有这两个属性，还有优先级，那么三个如何联系起来呢？嗯...可以接着往下看。

接下来，我们来表示词法单元的位置信息。我使用了一个单独的类来表示，名叫TokenLocation，该类具有三个属性：源文件名字，行号，列号。而我们看这个类，可以发现，该类还有一个方法名叫toString，该类的作用在于输出信息：文件名，行号与列号。而该类的名字其实是来自Java的启发，我相信熟悉Java的朋友一定不会陌生这个方法。

到此为止，我们已经只剩下最后一个词法单元属性了，即词法单元优先级。而这个则很简单，只是一个int而已。只是这个int数值需要与TokenType，TokenValue绑定在一起，那么如何绑定呢？答案是std::tuple。tuple是C++11标准库引入的一个新容器，在C++11之前，我们有map可以将两个放在一起，然而面对三个及其以上的元素则没有标准库容器对应，而tuple则是为了解决这个问题而引入的。有了tuple，我们就可以把三个及其以上的元素绑定在一起，那么即可解决我们现有的问题。而有关tuple更多的用法与信息，可以点击这个[链接][7]进行参考，有同学可能会抑或这个tuple注册在了哪里？似乎并没有在token.h。而这个tuple其实注册在了一个名叫dictionary的地方，而dictionary包含两个文件，分别是dictionary.h与dictionary.cpp。对于dictionary来说，其构造的时候，会把所有的相关Token都注册在dictionary里面，比如我们现在有一个名字叫”and”的，那么我们就会把”and”及其相关的词法单元类型，词法单元值，词法单元符号优先级都注册在dictionary里面。那么如何又把”and”与含有词法三属性的tuple绑定在一起呢？答案就是map。所以dictionary将会具有一个std::string, std::tuple<TokenValue, TokenType, int>的map。而为什么叫dictionary呢？我们再次类比英文单词，我们有意义的单词都存在于了英文字典当中，那么我们在查询butterfly这个单词的相关信息的时候，也是依照字典里面butterfly的含义，如butterfly是名词，以及它是蝴蝶的意思等等。所以，在我的编译世界认知中，我认为也有一个字典，我将其取名为dictionary，它不仅有添加Token的addToken方法，也有进行查询匹配关键字的lookup方法，更有查询这个Token是否包含在dictionary的haveToken方法。而这两个方法异常简单，我在这儿也就不细说了，具体的可以参考代码，如果有不懂的，可以在文章下面评论。

现在，我们已经完成了词法单元的五属性，那么我们就可以构造整个词法单元了，那么这个表示则是使用一个名叫Token的class来表示。而Token类则理所当然的包含了词法单元的五属性，并且加上了用于存储整形数值常量、字符串数值常量、浮点数数值常量的数据成员，而构造函数也一一对应。那么我们这儿可以发现整形数值常量等三个常量的构造函数没有包含Token的优先级，为什么不需要呢？其实答案很简单，因为这三个常量是确定的没有优先级，我们不需要源文件来指定，所以不需要再如同Token的第二个构造函数一样指定优先级，在它们构造的时候，只需要直接赋予其优先级为-1即可。而Token还有一些辅助的用于获取词法单元信息的getXXX方法以及用于输出的dump方法等，我相信读者应该可以自行读懂，这个并没有难度。而这里稍微提一句的是，现在的dump我们其实是在main.cpp那里直接调用输出的，这自然不符合编译器通过选项控制的特点。于是，在后续的时候，我们在装配编译器的时候，我们会通过编译器控制选项-v来进行调用这个方法，以达到debug输出的目的，与此同理的是，在输入测试文件的时候，现在是硬编码scanner_test.pas作为输入，而后续也会设定argv\[2\]这样的控制台参数来达到输入，但是我们目前这个阶段还并不需要这样，但是我们在最后装配整合编译器的时候，我们会做的，会有-O1，-O2，-O3，-v = 1这样的控制选项来控制，但是让我们一步一步的来. :-)

在我们完成了词法单元的表示，以及注册到dictionary以后，万事俱备，只欠词法分析器这个驱动了。而词法分析器则分布在了scanner.h和scanner.cpp中，使用class Scanner表示。而这个类粗看一下有很多方法和属性，但是其最重要的只有那么几个而已。首先，对于Scanner来说，其含有一个重要的数据成员：enum class State. 回顾我们第二节的原理课，我们知道词法分析器的根本在于状态机，而状态机的运转则是在一个又一个的状态进行切换，而我们的State则是状态机的状态，包括了IDENTIFIER, END_OF_FILE, NUMBER等状态。那么回顾我们第二节的状态图，我们发现有两个重要的状态：起始和结束。而在我们的状态表示中，其实是一个状态，名为NONE，为什么可以这么说呢？其实在我们从NONE开始起跳，然后完成一个词法单元的识别，添加一个Token以后，那么也就是END状态了，而这个END状态则又刚好是另一个词法单元的起始状态了，所以它其实一个状态，也就是我们这里的NONE. 而除了这个数据成员，我们也看见了熟悉的Token，TokenLocation，Dictionary等信息，同时还有一些新面孔，如errorFlag\_，这个成员则是指示词法分析器是否遇到了词法分析错误，如不正确的浮点数表示，找不到源文件等，其初始值为false，一旦遇到了错误则变为true，并且会有errorToken方法来进行词法分析错误信息输出，而这个方法则存在于了error.h中。我们的编译器与商业编译器的不同点则在于了错误处理方面其实是非常简单的，有时候甚至是考虑不全面的。而错误处理信息输出其实也是一个编译器的重要体现，Clang鄙视GCC的一个地方就是GCC的错误信息输出很弱，尤其是C++的模板输出信息，犹如天书。当然GCC也知耻而后勇，正在努力赶上（不过某些公司似乎非常喜欢GCC 3.4.5什么的，我也不清楚为什么喜欢，也许大概咳咳咳......）。而除了errorFlag\_，buffer\_等这样的辅助成员，还有一个重要的成员currentChar_，该成员则指示着我们当前词法分析器读的字符是什么，而与之相关的方法有两个：getNextChar和peekChar。在getNextChar方法中，我们从源文件中读取一个字符，赋予给currentChar\_，同时进行简单的监测，如果遇到的是’\n’，则把行号加1，并且列号归0，而没有遇到的话，则列号加1，我想这也是很好理解的。而peekChar这个方法是非常重要的，我认为peek思想也是非常美妙的。我们有时候并不想把某个字符拿下来，我们只是想“窥视”一下这个字符，看是不是我们满意的。如果满意，我们就拿下来，如果不满意，我们就不要了，也没有任何损失，如我们有 : （冒号），但是有可能接下来的字符是 = （等号）,它们可以组成 := 这个赋值符号，那么我们在遇到 : 的时候，就可以去peek一下，看是不是我们想要的。

而我们Scanner的核心实现方法则是getNextToken，这是整个词法分析器的核心所在。虽然这个方法很短，但是却是那么的重要与美妙。这个方法包含着一个运转着的状态机，不断的进行Token匹配。首先，该方法含有一个是否匹配的bool变量matched，该变量为false，然后包含一个do...while循环运转状态机。这个状态机最开始为NONE，然后在状态机变为END_OF_FILE, IDENTIFIER等时，则进行handleXXX的处理方式，同时matched变为true。而在NONE的时候，我们则会进行一系列的运转，来进行词法匹配。首先，我们会有一个名叫preprocess的处理方法，该方法做一些预处理的活，包括了去除空白，以及消除注释等。而这里的消除注释包含两个：块注释与行注释。而Pascal的块注释是{...}，行注释是(\*...\*)，那么如果是Python的#注释，是否依然同理呢？那么如果是C/C++的 // 注释，/\* ... \*/是否依然是同理呢？我相信聪明的读者看了代码实现后一定知道如何实现。当然后面我也会提出一些问题，希望聪明的读者来解答。而在完成预处理后，那么我们就会进行状态匹配，比如我们当前读到的字符是数字型的，那么我们的状态就跑到了NUMBER，而如果是alpha，则会跑到IDENTIFIER等。而这里需要的地方则是NUMBER包括了很多种，有整型（十进制与十六进制）、浮点型（包括了科学型）等，那么我们在代码实现的时候，就需要分别处理，而这里的处理依然运用了状态机，名叫NumberState，而思想与getNextToken一模一样，我就不赘述了，具体的可以看代码实现。所以，我们可以发现，状态机的思想是多么的重要。而这里，我们只做了十进制与十六进制，那么我在代码里注释了一段话，也许你希望支持八进制，那么我相信你也能轻易做到 :-) 。而这里我们的Pascal十六进制是$1234f这样的形式，而C++则是0x1234f这样的形式，那么我们怎么处理呢？聪明的读者想到了吗？回想我们讲的peek思想哦，当我们遇到0的时候，我们只需要peekChar一下，看是否是x，不就知道是否是十六进制了么？那么如果是八进制呢？不也很简单么？我们同样peekChar一下，看是否是数字即可，而怎么判定，isdigit方法是你的好帮手（当然你也可以使用暴力的判定方式）。所以，即使我实现的是Pascal，只要你懂了，我愿意相信，你实现其它语言的也不会是难事。:-)

而在这儿还有一个重要的方法是handleIdentifer，在这里我们将会进行关键字与Identifier的区分，而我们怎么区分呢？那么我们的Dictionary就上场了。当遇到一串字符，我们不知道是不是关键字的时候，我们就去查字典，字典会告诉我们的。那么就调用我们已经预设好的lookup，然后得到Token的tuple信息，然后取出它的TokenType，则可以判定是Keywords还是Identifier了，而这里需要注意的一点儿是Pascal不区分大小写，那么我们则需要把字符转换为小写，那么我们只需要使用std::transform(buffer\_.begin(), buffer\_.end(), buffer\_.begin(), std::tolower);即可，而如果你想要让Pascal大小写敏感，则把这儿改掉即可。那么如果你想加关键字或者修改为其它语言的关键字，那么只需要修改Dictionary里面的预设信息即可，一切都是那么行云流水与自然，关键在于我们理解了，那么一切都是那么的轻松。

剩下的一些方法则没有难度了，比如addToBuffer啊，reduceBuffer啊，我相信你都能理解了（如果真的不理解，可以在文章评论留言），剩下的我们则是写一个main来驱动测试，那么就是不断的读取Token，直到读到末尾......

接下来，我提一些小问题给读者：
我们在读取数值类型的，比如 +12, -12的时候，我们读取出来的是+, 12或者-, 12，也就是说我们没有合并在一起，那么这对么？
我们在读取123and这样的Token的时候，我们是读取123，然后是and，也就是分开了，这样对么？
问一个简单的C++问题, 这也是我突然想到的，在handleXXX的方法中，我们都有loc_ = getTokenLocation()，然而我们的TokenLocation却并没有实现赋值操作符，那么我想问的是，我们的class都会默认生成拷贝构造函数和赋值操作符么？那么如果我们TokenLocation如果实现了move构造和move赋值操作符，那么在这里会调用move版本的赋值操作符么？（对这个问题感兴趣的读者可以去看看Lippman大师的深入C++对象模型这本书，写的很不错，不过这本书的对象模型确实已经有点儿老了，如果我有时间，我会写一个更新的对象模型分析出来，比如GCC的）。

现在，虽然我们实现了词法分析器，但是这只是起步，比如我们如何识别自定义类型、File类型、指针、数组等类型呢？我们如何进行语句的识别、代码生成、代码优化呢？这都是后续教程的内容，虽然路很漫漫，但是我们一起来走，一起来体会编译器的魅力与美丽！










[1]:https://github.com/FrozenGene/LLVMPascalCompiler
[2]:http://www.pascal-central.com/docs/iso7185.pdf
[3]:http://www2.informatik.uni-halle.de/lehre/pascal/sprache/gpc/gpc-lang.html
[4]: http://en.wikibooks.org/wiki/Pascal_Programming/
[5]:http://www.taoyue.com/tutorials/pascal
[6]:http://google-styleguide.googlecode.com/svn/trunk/cppguide.xml
[7]:http://en.cppreference.com/w/cpp/utility/tuple
