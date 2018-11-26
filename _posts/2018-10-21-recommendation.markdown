---
layout: post
title:  "推荐系统的常见算法"
date:   2018-10-21 11:49:45 +0200
categories: 推荐 算法
---
协同过滤是推荐算法中最基本的算法，主要分为基于用户的协同过滤算法和基于物品的协同过滤算法。

#### 基于用户的协同过滤算法
简单来说，要给用户u作推荐，那么只要找出那些和u之前的行为类似的用户，即和u比较像的用户，把他们的行为推荐给用户u即可。所以基于用户的系统过滤算法包括两个步骤：

- 1）找到和目标用户兴趣相似的用户集合
- 2）找到这个集合中的用户喜欢的，且目标用户没有听说过的物品推荐给目标用户。


第一步的关键点在于计算用户之间的相似度，相似度一般通过Jaccard公式或者余弦相似度即可求得，，所以目前而言，计算用户相似度的复杂度是O（N*N）, N为用户数量，在用户数比较大的网站中不实用，比如亚马逊用户数量肯定N>100000，那么这样的复杂度是不可接受的。

第一步时间复杂度的改进方法1：因为很多用户间其实相似度是为0的，如果看成是一个N*N的矩阵的话，肯定是个稀疏矩阵，那么我们其实没有必要浪费计算量在这些0上。我们可以建立物品到用户的倒查表，及可以根据物品找到所有对该物品有过行为的用户，然后遍历各物品，对一个物品然后找到对该物品有过行为的用户，然后计算这些用户间的行为相似度（共有行为+1，同时计算这些用户的行为数），最后计算两用户间的公有行为占各自行为的比重。

第一步时间复杂度的改进方法2：采用分布式系统来进行计算，因为计算该相似矩阵相互不影响，完全可以充分利用多台机器的计算能力

第一步计算相似度的改进方法：举个例子：如果两人都买过《新华辞典》，并不能说明这两人想像，因为这本书基本上人人都会买，而如果这两人都买过《机器学习》，那么我们可以肯定，这两人在这方面有相同的兴趣爱好，也就是说，越是对冷门物品有同样的行为，就越说明用户的相似性，即在计算用户相似性的时候，需要降低热门物品的影响（通过计算流行度来实现，然后用1/N(i)来计算公共行为比重，N(i)表示流行度，这样，流行度高的物品所占比重就比较低）

第二步则比较简单，选出K个和用户u最相似的用户，把他们喜欢过的物品并且用户u没有喜欢过的物品推荐给u即可。这里面K的选择非常重要。K越大，推荐的结果就越热门，流行度就越高，同时覆盖率越低，因为基本推荐的都是流行的物品.

第二步评分预测改进方法：一般来说并不是所有第二步中的物品都会推荐给用户，因为这样的物品还是非常多的，一般来说我们会选择topN, 选用户可能最感兴趣的N个商品。那么要选择前N个商品，肯定是根据评分来进行排序，这样便会遇到一个问题，不同人的评分基点不同，比如A评分基点在4，好看的电影评5分，不好看的评3分，但是B基点是2，好看的评3分，不好看的评1分，这样的话直接根据评分来计算是不精确的，改进方法是计算用户在基点上的评分，如A对好看的电影给了（5-4）分，对不好看的电影给了（3-4）分，B对好看的电影给了（3-2）分，对不好看的电影给了（1-2）分，这样来看其实两者对电影的评价是类似的，而在计算需推荐用户对电影的评分时，只需要计算邻域的均值加上该用户的基点（一般用平均值来算）

基于用户的协同过滤算法在实际应用得比较少，一方面是因为用户多了，算法的复杂度还是很高，另一方面是这样的推荐很难给出推荐理由，故一般工业界都选择基于物品的协同过滤算法。


### 基于物品的协同过滤算法
是业界应用最多的算法，主要思想是利用用户之前有过的行为，给用户推荐和之前物品类似的物品。

基于物品的协同过滤算法主要分为两步：

- 1）计算物品之间的相似度。
- 2）根据物品的相似度和用户的历史行为给用户生成推荐列表。
第一步的关键点在于计算物品之间的相似度，这里并不采用基于内容的相似性，而是去计算在喜欢物品i的用户中有多少是喜欢物品j的，这样计算的前提是用户的兴趣爱好一般是比较确定的，不容易变，那么当一个用户对两个物品都喜欢的时候，我们往往可以认为这两个物品可能属于同一分类。令N(i)表示购买物品i的用户数，则物品i和物品j的相似度可以用wij = N(i)&N(j)/N(i)来计算。

第一步时间复杂度的改进方法：和UserCF类似，我们可以建立一张用户-物品的倒查表，这样每次去计算一个用户有过行为的那些物品间的相似度，能够保证计算的相似度都是有用的，而不用花大的计算量在那些0上面（肯定是个稀疏矩阵）

第一步相似度的改进方法1：若根据上面的公式来计算相似度，你会发现，物品i跟流行物品j的相似度很高，因为流行读高，所以基本人人都会买，这样的话流行度高的物品就比较没有区分度，所以我们需要惩罚流行物品j的权重 wij = N(i)&N(j)/sqrt(N(i)×N(j))

第一步相似度的改进方法2：需要惩罚用户的活跃度。若用户活跃度比较低，只买了有限的几本书，那么这几本书很有可能在一个或者两个兴趣范围内，对计算物品相似度比较有用，但是如果说一书店卖家趁着打折把亚马逊90%的书都买了然后赚差价，那么该用户的行为对计算物品相似度就没什么作用，因为90%的书肯定会覆盖很多范围，故应该像改进方法一中惩罚用户的活跃度。

第一步相似度的改进方法3：物品相似度的归一话。归一化不仅仅能提高推荐的准确度，还可以提高推荐的覆盖率和多样性。比如亚马逊上，用户的兴趣爱好肯定是分成几类的，很少说爱好集中在一类。假设有两类A和B，A类是关于散文的的，B类是关于计算机的（假设用户是个文艺的程序员）, 当用户买了5本A类的书和5本B类的书后，我们要给用户来推荐书，如果按照之前的方法，最后按照相似度排序，很有可能计算机方面的书都非常相似，相似度都在0.8以上，而散文类的书则不那么相似，最高也就0.7，那么推荐的应该都会是B类物品，因为就算B类中排名比较低，但照样比A类要高，所以应该对相似度矩阵进行归一化，对每个物品i,取其最相似的物品j，设其相似度为1，其他物品(k)与i的相似度则变成sim(i,k)/sim(i,j)，这样的话排序后的推荐A，B类商品都有，就大大提高了准确度，覆盖率和多样性。

第二步则比较简单，计算物品与用户已买物品的相似度（权重和），然后根据相似度排序选出topN.

ItemCF在实际系统中运用的比较多，主要有两个优点：

- 1）item-item表相比如user-user表要小的多，处理起来比较容易
- 2）itemCF容易提供推荐理由，比如给你推荐《机器学习》是因为你之前买过《数据挖掘》，这样能增加信任度，提高用户和推荐系统的交互，进一步增强个性化推荐
比较上述两个算法可以看出，UserCF是推荐用户所在兴趣小组中的热点，更注重社会化，而ItemCF则是根据用户历史行为推荐相似物品，更注重个性化。所以UserCF一般用在新闻类网站中，如Digg,而ItemCF则用在其他非新闻类网站中，如Amazon,hulu等等。

因为在新闻类网站中，用户的兴趣爱好往往比较粗粒度，很少会有用户说只看某个话题的新闻，往往某个话题也不是天天会有新闻的。个性化新闻推荐更强盗新闻热点，热门程度和时效性是个性化新闻推荐的重点，个性化是补充，所以UserCF给用户推荐和他有相同兴趣爱好的人关注的新闻，这样在保证了热点和时效性的同时，兼顾了个性化。另外一个原因是从技术上考虑的，作为一种物品，新闻的更新非常快，而且实时会有新的新闻出现，而如果使用ItemCF的话，需要维护一张物品之间相似度的表，实际工业界这表一般是一天一更新的，这在新闻领域是万万不能接受的。

但是，在图书，电子商务和电影网站等方面，ItemCF则能更好的发挥作用。因为在这些网站中，用户的兴趣爱好一般是比较固定的，而且相比于新闻网站更细腻。在这些网站中，个性化推荐一般是给用户推荐他自己领域的相关物品。另外，这些网站的物品数量更新速度不快，一天一次更新可以接受。而且在这些网站中，用户数量往往远远大于物品数量，从存储的角度来讲，UserCF需要消耗更大的空间复杂度，另外，ItemCF可以方便的提供推荐理由，增加用户对推荐系统的信任度，所以更适合这些网站。


###  SVD: Singular Value Decomposition
主要参考论文《A Guide to Singular Value Decomposition for Collaborative Filtering》

其实一开始是比较疑惑的，因为一开始没有查看论文，只是网上搜了一下svd的概念和用法，搜到的很多都是如下的公式：
![png1]({{ site.baseurl }}/assets/img/recommendation/1.png){: .center-image }

其中假设C是m*n的话，那么可以得到三个分解后的矩阵，分别为m*r,r*r,r*n,这样的话就可以大大降低存储代价，但是这里特别需要注意的是：这个概念一开始是用于信息检索方面的，它的C矩阵式完整的，故他们可以直接把这个矩阵应用svd分解，但是在推荐系统中，用户和物品的那个评分矩阵是不完整的，是个稀疏矩阵，故不能直接分解成上述样子，也有的文章说把空缺的补成平均值等方法，但效果都不咋滴。。。

在看了上述论文之后，才明白，在协同过滤中应用svd，其实是个最优化问题，我们假设用户和物品之间没有直接关系，但是定义了一个维度，称为feature，feature是用来刻画特征的，比如描述这个电影是喜剧还是悲剧，是动作片还是爱情片，而用户和feature之间是有关系的，比如某个用户必选看爱情片，另外一个用户喜欢看动作片，物品和feature之间也是有关系的，比如某个电影是喜剧，某个电影是悲剧，那么通过和feature之间的联系，我们可以把一个评分矩阵rating = m*n(m代表用户数，n代表物品数)分解成两个矩阵的相乘：user_feature*T(item_feature), T()表示转置，其中user_feature是m*k的（k是feature维度，可以随便定），item_feature是n*k的，那么我们要做的就是求出这两者矩阵中的值，使得两矩阵相乘的结果和原来的评分矩阵越接近越好。

这里所谓的越接近越好，指的是期望越小越好，期望的式子如下：
![png2]({{ site.baseurl }}/assets/img/recommendation/2.png){: .center-image }

其中n表示用户数目，m表示物品数目，I[i][j]是用来表示用户i有没有对物品j评过分，因为我们只需要评过分的那些越接近越好，没评过的就不需要考虑，Vij表示训练数据中给出的评分，也就是实际评分，p(Ui,Mj)表示我们对用户i对物品j的评分的预测，结果根据两向量点乘得到，两面的两项主要是为了防止过拟合，之所以都加了系数1/2是为了等会求导方便。

这里我们的目标是使得期望E越小越好，其实就是个期望最小的问题，故我们可以用随机梯度下降来实现。随机梯度下降说到底就是个求导问题，处于某个点的时候，在这个点上进行求导，然后往梯度最大的反方向走，就能快速走到局部最小值。故我们对上述式子求导后得：
![png3]({{ site.baseurl }}/assets/img/recommendation/3.png){: .center-image }

所以其实这个算法的流程就是如下过程（5和6指上述的求导）：
![png4]({{ site.baseurl }}/assets/img/recommendation/4.png){: .center-image }

实现起来还是比较方便快捷的，这里rmse是用来评测效果的，后面会再讲。

上述算法其实被称为批处理式学习算法，之所以叫批处理是因为它的期望是计算整个矩阵的期望（so big batch），其实还存在增量式学习算法，批处理和增量式的区别就在于前者计算期望是计算整个矩阵的，后者只计算矩阵中的一行或者一个点的期望，其中计算一行的期望被称为不完全增量式学习，计算一个点的期望被称为完全增量式学习。

不完全增量式学习期望如下（针对矩阵中的一行的期望，也就是针对一个用户i的期望）：
![png5]({{ site.baseurl }}/assets/img/recommendation/5.png){: .center-image }
那么求导后的式子如下：
![png6]({{ site.baseurl }}/assets/img/recommendation/6.png){: .center-image }
算法的大致思想如下（9和10指上面的求导）：
![png7]({{ site.baseurl }}/assets/img/recommendation/7.png){: .center-image }
完全增量式学习算法是对每一个评分进行期望计算，期望如下：
![png8]({{ site.baseurl }}/assets/img/recommendation/8.png){: .center-image }
求导后如下：
![png9]({{ site.baseurl }}/assets/img/recommendation/9.png){: .center-image }
所以整个算法流程是这样的（13和14指上面的求导）：
![png10]({{ site.baseurl }}/assets/img/recommendation/10.png){: .center-image }

上述都是svd的变种，只不过实现方式不一样，根据论文所说，其中第三种完全增量式学习算法效果最好，收敛速度非常快。

当然还有更优的变种，考虑了每个用户，每个物品的bias,这里所谓的bias就是每个人的偏差，比如一个电影a,b两人都认为不错，但是a评分方便比较保守，不错给3分，b评分比较宽松，不错给4分，故一下的评分方式考虑到了每个用户，每个物品的bias，要比上述算法更加精准。原来评分的话是直接计算user_feature*T(item_feature), T()表示转置，但现在要考虑各种bias，如下：
![png11]({{ site.baseurl }}/assets/img/recommendation/11.png){: .center-image }

其中a表示所有评分的平均数，ai表示用户i的bias，Bj表示物品j的偏差，相乘的矩阵还是和上面一样的意思。

故这时候的期望式子和求导的式子如下（这里只写了bias的求导，矩阵求导还是和上面一样）：
![png12]({{ site.baseurl }}/assets/img/recommendation/12.png){: .center-image }

当然了，光说不练假把式，我们选择了最后一种算法，及考虑bias的算法来实现了一把，数据源是来自movielens的100k的数据，其中包含了1000个用户对2000件物品的评分（当然，我这里是直接开的数组，要是数据量再大的话，就不这么实现了，主要是为了验证一把梯度下降的效果），用其中的base数据集来训练模型，用test数据集来测试数据，效果评测用一下式子来衡量：
![png13]({{ site.baseurl }}/assets/img/recommendation/13.png){: .center-image }

说白了就是误差平方和。。。同时我们也记录下每一次迭代后训练数据集的rmse.

代码如下：
{% highlight c++ %}
#include <iostream>  
#include <string>  
#include <fstream>  
#include <math.h>  
using namespace std;  
const int USERMAX = 1000;  
const int ITEMMAX = 2000;  
const int FEATURE = 50;  
const int ITERMAX = 20;  
double rating[USERMAX][ITEMMAX];  
int I[USERMAX][ITEMMAX];//indicate if the item is rated  
double UserF[USERMAX][FEATURE];  
double ItemF[ITEMMAX][FEATURE];  
double BIASU[USERMAX];  
double BIASI[ITEMMAX];  
double lamda = 0.15;  
double gamma = 0.04;  
double mean;  

double predict(int i, int j)  
{  
    double rate = mean + BIASU[i] + BIASI[j];  
    for (int f = 0; f < FEATURE; f++)  
        rate += UserF[i][f] * ItemF[j][f];  

    if (rate < 1)  
        rate = 1;  
    else if (rate>5)  
        rate = 5;  
    return rate;  
}  

double calRMSE()  
{  
    int cnt = 0;  
    double total = 0;  
    for (int i = 0; i < USERMAX; i++)  
    {  
        for (int j = 0; j < ITEMMAX; j++)  
        {  
            double rate = predict(i, j);  
            total += I[i][j] * (rating[i][j] - rate)*(rating[i][j] - rate);  
            cnt += I[i][j];  
        }  
    }  
    double rmse = pow(total / cnt, 0.5);  
    return rmse;  
}  
double calMean()  
{  
    double total = 0;  
    int cnt = 0;  
    for (int i = 0; i < USERMAX; i++)  
        for (int j = 0; j < ITEMMAX; j++)  
        {  
            total += I[i][j] * rating[i][j];  
            cnt += I[i][j];  
        }  
    return total / cnt;  
}  
void initBias()  
{  
    memset(BIASU, 0, sizeof(BIASU));  
    memset(BIASI, 0, sizeof(BIASI));  
    mean = calMean();  
    for (int i = 0; i < USERMAX; i++)  
    {  
        double total = 0;  
        int cnt = 0;  
        for (int j = 0; j < ITEMMAX; j++)  
        {  
            if (I[i][j])  
            {  
                total += rating[i][j] - mean;  
                cnt++;  
            }  
        }  
        if (cnt > 0)  
            BIASU[i] = total / (cnt);  
        else  
            BIASU[i] = 0;  
    }  
    for (int j = 0; j < ITEMMAX; j++)  
    {  
        double total = 0;  
        int cnt = 0;  
        for (int i = 0; i < USERMAX; i++)  
        {  
            if (I[i][j])  
            {  
                total += rating[i][j] - mean;  
                cnt++;  
            }  
        }  
        if (cnt > 0)  
            BIASI[j] = total / (cnt);  
        else  
            BIASI[j] = 0;  
    }  
}  
void train()  
{  
    //read rating matrix  
    memset(rating, 0, sizeof(rating));  
    memset(I, 0, sizeof(I));  
    ifstream in("ua.base");  
    if (!in)  
    {  
        cout << "file not exist" << endl;  
        exit(1);  
    }  
    int userId, itemId, rate;  
    string timeStamp;  
    while (in >> userId >> itemId >> rate >> timeStamp)  
    {  
        rating[userId][itemId] = rate;  
        I[userId][itemId] = 1;  
    }  
    initBias();  

    //train matrix decomposation  
    for (int i = 0; i < USERMAX; i++)  
        for (int f = 0; f < FEATURE; f++)  
            UserF[i][f] = (rand() % 10)/10.0 ;  
    for (int j = 0; j < ITEMMAX; j++)  
        for (int f = 0; f < FEATURE; f++)  
            ItemF[j][f] = (rand() % 10)/10.0 ;  

    int iterCnt = 0;  
    while (iterCnt < ITERMAX)  
    {  
        for (int i = 0; i < USERMAX; i++)  
        {  

            for (int j = 0; j < ITEMMAX; j++)  
            {  
                if (I[i][j])  
                {  

                    double predictRate = predict(i, j);  
                    double eui = rating[i][j] - predictRate;  
                    BIASU[i] += gamma*(eui - lamda*BIASU[i]);  
                    BIASI[j] += gamma*(eui - lamda*BIASI[j]);  
                    for (int f = 0; f < FEATURE; f++)  
                    {  
                        UserF[i][f] += gamma*(eui*ItemF[j][f] - lamda*UserF[i][f]);  
                        ItemF[j][f] += gamma*(eui*UserF[i][f] - lamda*ItemF[j][f]);  
                    }                 
                }  

            }  

        }  
        double rmse = calRMSE();  
        cout << "Loop " << iterCnt << " : rmse is " << rmse << endl;  
        iterCnt++;  
    }  

}  

void test()  
{  
    ifstream in("ua.test");  
    if (!in)  
    {  
        cout << "file not exist" << endl;  
        exit(1);  
    }  
    int userId, itemId, rate;  
    string timeStamp;  
    double total = 0;  
    double cnt = 0;  
    while (in >> userId >> itemId >> rate >> timeStamp)  
    {  
        double r = predict(userId, itemId);  
        total += (r - rate)*(r - rate);  
        cnt += 1;  
    }  
    cout << "test rmse is " << pow(total / cnt, 0.5) << endl;  
}  
int main()  
{  
    train();  
    test();  
    return 0;  
}

{% endhighlight %}
效果图如下：
![png14]({{ site.baseurl }}/assets/img/recommendation/14.png){: .center-image }

可以看到，rmse能非常快的收敛，训练数据中的rmse能很快收敛到0.8左右，然后拿测试集的数据去测试，rmse为0.949，也是蛮不错的预测结果了，当然这里可以调各种参数来获得更优的实验结果。。。就是所谓的黑科技？。。。：）

### SlopeOne算法
SlopeOne算法是一个非常简单的协同过滤算法，主要思想如下：如果用户u对物品j打过分，现在要对物品i打分，那么只需要计算出在同时对物品i和j打分的这种人中，他们的分数之差平均是多少，那么我们就可以根据这个分数之差来计算用户u对物品i的打分了，当然，这样的物品j也有很多个，那有的物品和j共同打分的人少，有的物品和j共同打分的人多，那么显而易见，共同打分多的那个物品在评分时所占的比重应该大一些。

如上就是简单的SlopeOne算法的主要思想，用维基百科上的一张图来表示（一看就懂）：
![png15]({{ site.baseurl }}/assets/img/recommendation/15.png){: .center-image }
途中用户B要对物品J进行评分，那么这时候发现物品i被用户B打为2分，而同时发现用户A同时评价了物品i和物品j,且物品i比物品j少了0.5分，那么由此看来，用户B给物品j打得分应该就是比给物品i打的分高0.5分，故是2.5分。

由于思想是如此简单，故我们就来实践一把，当然这里就是最最朴素的实现，只是为了检测下算法效果如何。。。数据集还是如上篇博客一样，用的是movielens里面的小数据集，其中有1000用户对2000物品的评分，80%用来训练，20%用来测试。

具体代码如下：
{% highlight c++ %}
#include <iostream>  
#include <string>  
#include <fstream>  
#include <math.h>  
using namespace std;  
const int USERMAX = 1000;  
const int ITEMMAX = 2000;  
double rating[USERMAX][ITEMMAX];  
int I[USERMAX][ITEMMAX];//indicate if the item is rated  
double mean;  

double predict(int u, int l)  
{  
    double total = 0;  
    double totalCnt = 0;  
    for (int i = 0; i < ITEMMAX; i++)  
    {  
        if (l != i&&I[u][i])  
        {  
            double dev = 0;  
            int cnt = 0;  
            for (int j = 0; j < USERMAX; j++)  
            {  
                if (I[j][l] && I[j][i])  
                {  
                    dev += rating[j][i]-rating[j][l];  
                    cnt++;  
                }  
            }  
            if (cnt)  
            {  
                dev /= cnt;  
                total += (rating[u][i] - dev)*cnt;  
                totalCnt += cnt;  
            }  
        }  
    }  
    if (totalCnt == 0)  
        return mean;  
    return total / totalCnt;  
}  
double calMean()  
{  
    double total = 0;  
    int cnt = 0;  
    for (int i = 0; i < USERMAX; i++)  
        for (int j = 0; j < ITEMMAX; j++)  
        {  
            total += I[i][j] * rating[i][j];  
            cnt += I[i][j];  
        }  
    return total / cnt;  
}  

void train()  
{  
    //read rating matrix  
    memset(rating, 0, sizeof(rating));  
    memset(I, 0, sizeof(I));  
    ifstream in("ua.base");  
    if (!in)  
    {  
        cout << "file not exist" << endl;  
        exit(1);  
    }  
    int userId, itemId, rate;  
    string timeStamp;  
    while (in >> userId >> itemId >> rate >> timeStamp)  
    {  
        rating[userId][itemId] = rate;  
        I[userId][itemId] = 1;  
    }     
    mean = calMean();  
}  

void test()  
{  
    ifstream in("ua.test");  
    if (!in)  
    {  
        cout << "file not exist" << endl;  
        exit(1);  
    }  
    int userId, itemId, rate;  
    string timeStamp;  
    double total = 0;  
    double cnt = 0;  
    while (in >> userId >> itemId >> rate >> timeStamp)  
    {  
        double r = predict(userId, itemId);  
        cout << "true: " << rate << " predict: " << r << endl;  
        total += (r - rate)*(r - rate);  
        cnt += 1;  
        //cout << total << endl;  
    }  
    cout << "test rmse is " << pow(total / cnt, 0.5) << endl;  
}  
int main()  
{  
    train();  
    test();  
    return 0;  
}
{% endhighlight %}

实验结果如下：
![png16]({{ site.baseurl }}/assets/img/recommendation/16.png){: .center-image }
在测试集上的rmse达到了0.96，而之前一篇博客实现的svd通过复杂的梯度下降来求最优解也就0.95左右，故SlopeOne算法是非常简单有效的，维基百科里说是最简洁的协同过滤了，但是我个人觉得类似knn的协同过滤更加好懂啊（只不过在计算用户相似度等方面麻烦了点）
