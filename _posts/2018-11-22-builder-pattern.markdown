---
layout: post
title:  "生成器模式(Builder Pattern)的应用"
date:   2018-11-22 11:49:45 +0200
categories: 设计模式, Builder-Pattern
---
经常我们的类会包含多个属性，构造这个类的对象时则会通过构造函数来构造，因此构造函数经常长的要命，尤其是随着需求的变化，构造函数发生改动，那么很多时候，我们既要支持原有的构造函数，又要支持新的参数，则这个类就会变的臃肿不堪，来看下面的例子，假设我们有这样一个类TestBuilderPattern,除了构造函数之外的函数均被省略，如下：
<pre><code class="language-java">
public class TestBuilderPattern {
    private int x1;
    private int x2;
    private int x3;
    private double y1;
    private double y2;
    private double y3;
    private String z1;
    private String z2;
    private String z3;

    public TestBuilderPattern(int x1, int x2, int x3, double y1, double y2, double y3, String z1, String z2, String z3) {
        this.x1 = x1;
        this.x2 = x2;
        this.x3 = x3;
        this.y1 = y1;
        this.y2 = y2;
        this.y3 = y3;
        this.z1 = z1;
        this.z2 = z2;
        this.z3 = z3;
    }
}
</code></pre>

假设现在有了新需求1，需要支持一个新的属性int x4,但是很多其他代码都已经用了上述的构造函数，则现在有两个选择，一是直接修改这个构造函数，并修改所有相应的调用，这个改动的影响会非常大，涉及很多个文件代码的改动，二是维持这个构造函数，并支持新的构造函数，如下：
<pre><code class="language-java">
public class TestBuilderPattern {
    private int x1;
    private int x2;
    private int x3;
    private int x4;
    private double y1;
    private double y2;
    private double y3;
    private String z1;
    private String z2;
    private String z3;

    public TestBuilderPattern(int x1, int x2, int x3, int x4, double y1, double y2, double y3, String z1, String z2, String z3) {
        this.x1 = x1;
        this.x2 = x2;
        this.x3 = x3;
        this.x4 = x4;
        this.y1 = y1;
        this.y2 = y2;
        this.y3 = y3;
        this.z1 = z1;
        this.z2 = z2;
        this.z3 = z3;
    }
    public TestBuilderPattern(int x1, int x2, int x3, double y1, double y2, double y3, String z1, String z2, String z3) {
        this(x1, x2, x3, 0, y1, y2, y3, z1, z2, z3);
    }
}
</code></pre>

一般我们通过保留支持所有参数的构造函数，让参数少的构造函数调用参数多的构造函数，并给予某些参数默认值，如上面的我们给予x4=0在参数少的构造函数中。

再假设现在新需求2又需要支持一个构造函数，但不需要x4,而需要另外一个参数String s4,则代码将会变成如下样子：
<pre><code class="language-java">
public class TestBuilderPattern {
    private int x1;
    private int x2;
    private int x3;
    private int x4;
    private double y1;
    private double y2;
    private double y3;
    private String z1;
    private String z2;
    private String z3;
    private String z4;

    public TestBuilderPattern(int x1, int x2, int x3, int x4, double y1, double y2, double y3, String z1, String z2, String z3, String z4) {
        this.x1 = x1;
        this.x2 = x2;
        this.x3 = x3;
        this.x4 = x4;
        this.y1 = y1;
        this.y2 = y2;
        this.y3 = y3;
        this.z1 = z1;
        this.z2 = z2;
        this.z3 = z3;
        this.z4 = z4;
    }

    public TestBuilderPattern(int x1, int x2, int x3, double y1, double y2, double y3, String z1, String z2, String z3, String z4) {
        this(x1, x2, x3, 0, y1, y2, y3, z1, z2, z3, z4);
    }

    public TestBuilderPattern(int x1, int x2, int x3, int x4, double y1, double y2, double y3, String z1, String z2, String z3) {
        this(x1, x2, x3, x4, y1, y2, y3, z1, z2, z3, "");
    }

    public TestBuilderPattern(int x1, int x2, int x3, double y1, double y2, double y3, String z1, String z2, String z3) {
        this(x1, x2, x3, 0, y1, y2, y3, z1, z2, z3, "");
    }
}
</code></pre>

可以看到，首先这种代码对变化来说不友好，当需要添加参数的时候，做的改动比较大，而且基本上是修改原有代码，其次这样的构造函数影响代码的可读性，这么多参数，并不知道这个参数到底对应着这个类的哪个属性，所以这时候就需要用构造者模式了(Builder Pattern),构造者模式相当于是提供了一个中间层，用这个中间层来控制对象的构造，如最一开始的9个参数的类，我们用构造者模式来实现一把，如下：
<pre><code class="language-java">
public class TestBuilderPattern {
    private int x1;
    private int x2;
    private int x3;
    private double y1;
    private double y2;
    private double y3;
    private String z1;
    private String z2;
    private String z3;

    private TestBuilderPattern(int x1, int x2, int x3, double y1, double y2, double y3, String z1, String z2, String z3) {
        this.x1 = x1;
        this.x2 = x2;
        this.x3 = x3;
        this.y1 = y1;
        this.y2 = y2;
        this.y3 = y3;
        this.z1 = z1;
        this.z2 = z2;
        this.z3 = z3;
    }

    public static final class Builder {
        private int x1 = 0;//better to provide default value
        private int x2 = 0;
        private int x3 = 0;
        private double y1 = 0;
        private double y2 = 0;
        private double y3 = 0;
        private String z1 = "";
        private String z2 = "";
        private String z3 = "";

        public Builder setX1(int x1) {
            this.x1 = x1;
            return this;
        }

        public Builder setX2(int x2) {
            this.x2 = x2;
            return this;
        }

        public Builder setX3(int x3) {
            this.x3 = x3;
            return this;
        }

        public Builder setY1(double y1) {
            this.y1 = y1;
            return this;
        }

        public Builder setY2(double y2) {
            this.y2 = y2;
            return this;
        }

        public Builder setY3(double y3) {
            this.y3 = y3;
            return this;
        }

        public Builder setZ1(String z1) {
            this.z1 = z1;
            return this;
        }

        public Builder setZ2(String z2) {
            this.z2 = z2;
            return this;
        }

        public Builder setZ3(String z3) {
            this.z3 = z3;
            return this;
        }

        public TestBuilderPattern Build() {
            return new TestBuilderPattern(x1, x2, x3, y1, y2, y3, z1, z2, z3);
        }
    }

    public static void main(String[] args) {
        TestBuilderPattern tbp = new TestBuilderPattern.Builder()
                .setX1(1)
                .setX2(2)
                .setX3(3)
                .setY1(1.1)
                .setY2(2.2)
                .setY3(3.3)
                .setZ1("z1")
                .setZ2("z2")
                .setZ3("z3")
                .Build();
    }
}
</code></pre>

可以看到，我们在TestBuilderPattern类里面加了一个内嵌的类，用来专门构造对象，然后把构造函数私有化，不让用户直接调用构造函数来构造对象，这样如何构造对象就会被我们隐藏起来，或者锁控制起来，在main函数的调用里，可以看到，我们使用一系列的set函数来设置属性，最后通过Build()构造对象，这样的代码，相比之前构造函数连续9个参数，更具有可读性。

下面来看看如果需要增加新需求1，和上面一样增加一个参数int x4,则使用构造者参数的代码如下：
<pre><code class="language-java">
public class TestBuilderPattern {
    private int x1;
    private int x2;
    private int x3;
    private int x4;
    private double y1;
    private double y2;
    private double y3;
    private String z1;
    private String z2;
    private String z3;

    private TestBuilderPattern(int x1, int x2, int x3, int x4, double y1, double y2, double y3, String z1, String z2, String z3) {
        this.x1 = x1;
        this.x2 = x2;
        this.x3 = x3;
        this.x4 = x4;
        this.y1 = y1;
        this.y2 = y2;
        this.y3 = y3;
        this.z1 = z1;
        this.z2 = z2;
        this.z3 = z3;
    }

    public static final class Builder {
        private int x1 = 0;//better to provide default value
        private int x2 = 0;
        private int x3 = 0;
        private int x4 = 0;
        private double y1 = 0;
        private double y2 = 0;
        private double y3 = 0;
        private String z1 = "";
        private String z2 = "";
        private String z3 = "";

        public Builder setX1(int x1) {
            this.x1 = x1;
            return this;
        }

        public Builder setX2(int x2) {
            this.x2 = x2;
            return this;
        }

        public Builder setX3(int x3) {
            this.x3 = x3;
            return this;
        }

        public Builder setX4(int x4) {
            this.x4 = x4;
            return this;
        }

        public Builder setY1(double y1) {
            this.y1 = y1;
            return this;
        }

        public Builder setY2(double y2) {
            this.y2 = y2;
            return this;
        }

        public Builder setY3(double y3) {
            this.y3 = y3;
            return this;
        }

        public Builder setZ1(String z1) {
            this.z1 = z1;
            return this;
        }

        public Builder setZ2(String z2) {
            this.z2 = z2;
            return this;
        }

        public Builder setZ3(String z3) {
            this.z3 = z3;
            return this;
        }

        public TestBuilderPattern Build() {
            return new TestBuilderPattern(x1, x2, x3, x4, y1, y2, y3, z1, z2, z3);
        }
    }

    public static void main(String[] args) {
        TestBuilderPattern tbp = new TestBuilderPattern.Builder()
                .setX1(1)
                .setX2(2)
                .setX3(3)
                .setX4(4)
                .setY1(1.1)
                .setY2(2.2)
                .setY3(3.3)
                .setZ1("z1")
                .setZ2("z2")
                .setZ3("z3")
                .Build();
    }
}
</code></pre>

可以看到，修改点主要在以下几方面：

<li>在TestBuilderPettern添加int x4属性</li>
<li>在Builder类里添加int x4属性，并增加set方法</li>

基本上修改部分都是新需求部分，不会修改原有代码（最多一个构造函数，这个是不可避免的）。这样的实现通过Builder类里面属性的默认值来支持原有的接口(原来的接口只比现在少了一个setX4函数，但因为x4有默认值，所以不影响使用)

再来看新需求2，不需要int x4了，需要string z4,则利用构造者模式的代码如下：

<pre><code class="language-java">
public class TestBuilderPattern {
    private int x1;
    private int x2;
    private int x3;
    private int x4;
    private double y1;
    private double y2;
    private double y3;
    private String z1;
    private String z2;
    private String z3;
    private String z4;

    private TestBuilderPattern(int x1, int x2, int x3, int x4, double y1, double y2, double y3, String z1, String z2, String z3, String z4) {
        this.x1 = x1;
        this.x2 = x2;
        this.x3 = x3;
        this.x4 = x4;
        this.y1 = y1;
        this.y2 = y2;
        this.y3 = y3;
        this.z1 = z1;
        this.z2 = z2;
        this.z3 = z3;
        this.z4 = z4;
    }

    public static final class Builder {
        private int x1 = 0;//better to provide default value
        private int x2 = 0;
        private int x3 = 0;
        private int x4 = 0;
        private double y1 = 0;
        private double y2 = 0;
        private double y3 = 0;
        private String z1 = "";
        private String z2 = "";
        private String z3 = "";
        private String z4 = "";

        public Builder setX1(int x1) {
            this.x1 = x1;
            return this;
        }

        public Builder setX2(int x2) {
            this.x2 = x2;
            return this;
        }

        public Builder setX3(int x3) {
            this.x3 = x3;
            return this;
        }

        public Builder setX4(int x4) {
            this.x4 = x4;
            return this;
        }

        public Builder setY1(double y1) {
            this.y1 = y1;
            return this;
        }

        public Builder setY2(double y2) {
            this.y2 = y2;
            return this;
        }

        public Builder setY3(double y3) {
            this.y3 = y3;
            return this;
        }

        public Builder setZ1(String z1) {
            this.z1 = z1;
            return this;
        }

        public Builder setZ2(String z2) {
            this.z2 = z2;
            return this;
        }

        public Builder setZ3(String z3) {
            this.z3 = z3;
            return this;
        }

        public Builder setZ4(String z4) {
            this.z4 = z4;
            return this;
        }

        public TestBuilderPattern Build() {
            return new TestBuilderPattern(x1, x2, x3, x4, y1, y2, y3, z1, z2, z3, z4);
        }
    }

    public static void main(String[] args) {
        TestBuilderPattern tbp = new TestBuilderPattern.Builder()
                .setX1(1)
                .setX2(2)
                .setX3(3)
                .setY1(1.1)
                .setY2(2.2)
                .setY3(3.3)
                .setZ1("z1")
                .setZ2("z2")
                .setZ3("z3")
                .setZ4("z4")
                .Build();
    }
}
</code></pre>

可以看到，新需求2的改动和需求1的改动一模一样，不会像最开始用构造函数时的变动一样产生多个中间构造函数，之后每增加一个变量都是一样的操作.

总结：构造者模式适用于构造函数较多，且参数较容易变化的类。采用构造者模式的类，不仅可以更好的应对需求变化，而且可以保证代码的可读性。或者说当构建对象的算法比较复杂时，我们可以把它单独抽象出来形成一个中间层，用来专门控制对象的生成。
