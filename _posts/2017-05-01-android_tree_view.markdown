---
layout: post
title:  利用RecyclerView打造高性能树形控件
date:   2017-05-01
---

<p class="intro"><span class="dropcap">前</span>段时间项目里需要大量使用树形结构的控件，由于开发周期的关系，第一时间去GitHub找了Star最多的库[https://github.com/bmelnychuk/AndroidTreeView]进行改造，该控件的原理很简单:View的结构和树形结构一致，每一个节点的View为一个LinearLayout,包含两个子View,第一个为自身，第二个为包裹子节点的ViewGroup。第一次显示TreeView的时候默认展开一级（当然也可以用expandAll方法展开全部），展开的逻辑就是遍历所有子节点，然后拿到ViewHolder进行添加，每一个节点持有一个ViewHolder,ViewHolder包含有View和渲染View的方法，第一次展开的时候根据ViewHolder渲染的方法创建出View，下次展开的时候就不用new了。好了，分析完这个控件之后来看看其中的优缺点，优点：结构简单，思路清晰，构造数据方便。缺点：性能差！性能差！性能差！当子节点数量达到50就能感觉到展开有明显卡顿，这一个缺点足以让我放弃了该控件，恰好这期迭代安排有变动，有一周时间用来重写这个部分。
</p>

 性能不好当然得用性能好的方式解决，RecyclerView就是我们的主角，把所有节点都视为RecyclerView的一个Item，这样就能做到无论树的节点数量多大，深度有多深，都能回收利用ItemView。经过一周的改造和优化，目前该控件基本能满足一般场景的树形控件需求。
先看效果：
<img src="{{ '/public/img/tree_demo.jpg' | prepend: site.baseurl }}" alt="">

#### 实现详解
先看基本类TreeNode：
{% highlight java %}
public class TreeNode {  
    private int level;  
  
    private Object value;  
  
    private TreeNode parent;  
  
    private List\<TreeNode\> children;  
  
    private int index;  
  
    private boolean expanded;  
  
    private boolean selected;  
  
    private boolean itemClickEnable = true;  
  
    public TreeNode(Object value) {  
        this.value = value;  
        this.children = new ArrayList\<\>();  
    }  
  
    public static TreeNode root() {  
        TreeNode treeNode = new TreeNode(null);  
        return treeNode;  
    }  
  
    public void addChild(TreeNode treeNode) {  
        if (treeNode == null) {  
            return;  
        }  
        children.add(treeNode);  
        treeNode.setIndex(getChildren().size());  
        treeNode.setParent(this);  
    }
//其他方法省略
}
{% endhighlight %}
树节点的基本样子，没啥好说的，只是要注意addChild的时候不光要add，还得顺便把child的parent赋值。接下来看TreeView类：
{% highlight java %}
public class TreeView implements SelectableTreeAction {  
    private TreeNode root;  
  
    private Context context;  
  
    private BaseNodeViewFactory baseNodeViewFactory;  
  
    private RecyclerView rootView;  
  
    private TreeViewAdapter adapter;  

//省略部分代码  
    public TreeView(@NonNull TreeNode root, @NonNull Context context,@NonNull BaseNodeViewFactory baseNodeViewFactory) {  
        this.root = root;  
        this.context = context;  
        this.baseNodeViewFactory = baseNodeViewFactory;  
        if (baseNodeViewFactory == null) {  
            throw new IllegalArgumentException("You must assign a BaseNodeViewFactory!");  
        }  
    }  
  
    public View getView() {  
        if (rootView == null) {  
            this.rootView = buildRootView();  
        }  
  
        return rootView;  
    }  
  
    @NonNull  
    private RecyclerView buildRootView() {  
        RecyclerView recyclerView = new RecyclerView(context);  
        //省略部分代码  
  
		recyclerView.setLayoutManager(new LinearLayoutManager(context));  
        adapter = new TreeViewAdapter(context, root, baseNodeViewFactory);  
        recyclerView.setAdapter(adapter);  
        return recyclerView;  
    }  
  
    @Override  
    public void expandAll() {  
        if (root == null) {  
            return;  
        }  
        TreeHelper.expandAll(root);  
  
        refreshTreeView();  
    }  
  
  
    private void refreshTreeView() {  
        if (rootView != null) {  
            ((TreeViewAdapter) rootView.getAdapter()).refreshView();  
        }  
    }  
  
    @Override  
    public void expandNode(TreeNode treeNode) {  
        adapter.expandNode(treeNode);  
    }  
  
    @Override  
    public void expandLevel(int level) {  
        TreeHelper.expandLevel(root, level);  
  
        refreshTreeView();  
    }

//其他方法省略
}
{% endhighlight %}
这是TreeView的直接操作类，外部的一切动作都通过TreeView转接。TreeView的创建必须传入一个BaseNodeViewFactory，这个工厂用来根据level得到每一级的BaseNodeViewBinder，稍后会分析这个类，然后创建一个RecyclerView作为rootView。TreeView包含了展开、收起、选择等全部的方法，但他只负责转接，具体实现细节全交给了TreeHelper和Adapter,那我们就先进入主角Adapter：
{% highlight java %}
public class TreeViewAdapter extends RecyclerView.Adapter {  
  
    private Context context;  
    private TreeNode root;  
    private List\<TreeNode\> expandedNodeList;  
	private BaseNodeViewFactory baseNodeViewFactory;  
	private View EMPTY_PARAMETER;  
  
    public TreeViewAdapter(Context context, TreeNode root,  
                           @NonNull BaseNodeViewFactory baseNodeViewFactory) {  
        this.context = context;  
        this.root = root;  
        this.baseNodeViewFactory = baseNodeViewFactory;  
  
        this.EMPTY_PARAMETER = new View(context);  
        this.expandedNodeList = new ArrayList\<\>();  
  
        buildExpandedNodeList();  
    }  
  
    private void buildExpandedNodeList() {  
        expandedNodeList.clear();  
  
        for (TreeNode child : root.getChildren()) {  
            insertNode(expandedNodeList, child);  
        }  
    }  
  
    private void insertNode(List\<TreeNode\> nodeList, TreeNode treeNode) {  
        nodeList.add(treeNode);  
  
        if (!treeNode.hasChild()) {  
            return;  
        }  
        if (treeNode.isExpanded()) {  
            for (TreeNode child : treeNode.getChildren()) {  
                insertNode(nodeList, child);  
            }  
        }  
    }  
  
    @Override  
    public int getItemViewType(int position) {  
        return expandedNodeList.get(position).getLevel();  
    }  
  
    @Override  
    public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int level) {  
        View view = LayoutInflater.from(context).inflate(baseNodeViewFactory  
                .getNodeViewBinder(EMPTY_PARAMETER, level).getLayoutId(), parent, false);  
  
        return baseNodeViewFactory.getNodeViewBinder(view, level);  
    }  
  
    @Override  
    public void onBindViewHolder(final RecyclerView.ViewHolder holder, final int position) {  
        final View nodeView = holder.itemView;  
        final TreeNode treeNode = expandedNodeList.get(position);  
        final BaseNodeViewBinder viewBinder = getNodeBinder(treeNode);  
//省略部分代码  
        if (viewBinder.getToggleTriggerViewId() != 0) {  
            View triggerToggleView = nodeView.findViewById(viewBinder.getToggleTriggerViewId());  
  
            if (triggerToggleView != null) {  
                triggerToggleView.setOnClickListener(new View.OnClickListener() {  
                    @Override  
                    public void onClick(View v) {  
                        onNodeToggled(treeNode);  
                        viewBinder.onNodeToggled(nodeView, treeNode, treeNode.isExpanded());  
                    }  
                });  
            }  
        }   
        viewBinder.bindView(nodeView, treeNode);  
    }  
    private void onNodeToggled(TreeNode treeNode) {  
        treeNode.setExpanded(!treeNode.isExpanded());  
  
        if (treeNode.isExpanded()) {  
            expandNode(treeNode);  
        } else {  
            collapseNode(treeNode);  
        }  
    }

public void expandNode(TreeNode treeNode) {  
    if (treeNode == null) {  
        return;  
    }  
    List\<TreeNode\> additionNodes = TreeHelper.expandNode(treeNode, false);  
    int index = expandedNodeList.indexOf(treeNode);  
  
    insertNodesAtIndex(index, additionNodes);  
}

public void collapseNode(TreeNode treeNode) {  
    if (treeNode == null) {  
        return;  
    }  
    List\<TreeNode\> removedNodes = TreeHelper.collapseNode(treeNode, false);  
    int index = expandedNodeList.indexOf(treeNode);  
  
    removeNodesAtIndex(index, removedNodes);  
}

//省略部分代码

@Override
public int getItemCount() {
return expandedNodeList == null ? 0 : expandedNodeList.size();
}
{% endhighlight %}
Adapter中用一个expandedNodeList存储当前可见的节点（包含屏幕外的节点），所以每次刷新之前都得重新计算这个集合，buildExpandedNodeList在首次或者变动较大的时候才用到，其他情况刷新局部数据会节省很多开销。接着分析adapter的两个关键方法onCreateViewHolder和onBindViewHolder： 
  onCreateViewHolder中根据BaseNodeViewBinder拿到layoutId，生成布局构BaseNodeViewBinder，这里是比较关键的部分，构造BaseNodeViewBinder必须传入View和Level，而View是用nodeViewBinder.getLayoutId()生成的，也就是要用的参数需要从结果中拿到，这里就形成了矛盾。仔细分析一下，这里我们只关注怎么拿到layoutId，只要成功构造出BaseNodeViewBinder完成任务，即使传入任意View也没有任何影响，因为真正的View是拿到layoutId之后生成的View，所以这里我们就传入了一个非空的new View(context)。
  onBindViewHolder承担了部分bind的职责，负责处理展开收起点击事件和选择逻辑，其他的细节由使用者实现。这里简单分析点击展开的逻辑，展开一个节点分为两步：首先拿到展开后新增需要显示的数据，接着调用notifyItemRangeInserted刷新局部。关键在于第一步，这一步的计算是在TreeHelper中进行，之前的TreeView中也多次用到这个类，TreeHelper其实就是纯负责计算的类，展开收起，添加删除，选择计算，脏活累活全交给了它，里面的具体算法细节就不多分析，感兴趣可以自行查看。

还有一个和Adapter紧密相关的BaseNodeViewBinder类，代码如下：

{% highlight java %}
public abstract class BaseNodeViewBinder extends RecyclerView.ViewHolder {  
  
    public BaseNodeViewBinder(View itemView) {  
        super(itemView);  
    }  
  
    public abstract int getLayoutId();
  
    public abstract void bindView(View view, TreeNode treeNode);  
  
    public int getToggleTriggerViewId() {  
        return 0;  
    }  
  
    public void onNodeToggled(View item, TreeNode treeNode, boolean expand) {  
        //empty  
    }  
}

{% endhighlight %}

BaseNodeViewBinder继承自ViewHolder，该类不光是一个ViewHolder,还包含了CreateViewHolder（拿到layoutId,具体过程还是在Adapter中进行），BindViewHolder的职责，可能你会觉得职责不够单一，但作为使用者来说清晰和简单就够了，因为使用者只用关心我的每一层级的节点布局是哪个、我该怎么把数据绑定到节点上，具体你这个类内部在干啥我根本不关注。另外还有两个不是必须实现的方法，getToggleTriggerViewId用来指定你想要点击触发展开收起操作的View，如果不指定默认是点击全部区域，当展开收起之后你需要做其他事就实现onNodeToggled方法。BaseNodeViewBinder还有一个子类CheckableNodeViewBinder，如果需要用到选择功能实现它就好了。

#### 如何使用
##### 添加依赖
{% highlight java %}
compile 'me.texy.treeview:treeview_lib:1.0.1'
{% endhighlight %}

##### 实现BaseNodeViewBinder
Sample：
{% highlight java %}
public class FirstLevelNodeViewBinder extends BaseNodeViewBinder {  
    public FirstLevelNodeViewBinder(View itemView) {  
        super(itemView);  
    }  
  
    @Override  
    public int getLayoutId() {  
        return R.layout.item_first_level;  
    }  
  
    @Override  
    public void bindView(View view, final TreeNode treeNode) {  
        TextView textView = (TextView) view.findViewById(R.id.node_name_view);  
        textView.setText(treeNode.getValue().toString());  
    }
}

SecondLevelNodeViewBinder
ThirdLevelNodeViewBinder
.
.
.
{% endhighlight %}

如果需要用选择功能则继承自CheckableNodeViewBinder
##### 实现BaseNodeViewFactory
Sample：
{% highlight java %}
public class MyNodeViewFactory extends BaseNodeViewFactory {  
  
    @Override  
    public BaseNodeViewBinder getNodeViewBinder(View view, int level) {  
        switch (level) {  
            case 0:  
                return new FirstLevelNodeViewBinder(view);  
            case 1:  
                return new SecondLevelNodeViewBinder(view);  
            case 2:  
                return new ThirdLevelNodeViewBinder(view);  
            default:  
                return null;  
        }  
    }  
}
{% endhighlight %}
##### 生成TreeView
Sample:
{% highlight java %}
TreeNode root = TreeNode.root();
//build the tree as you want
for (int i = 0; i \< 5; i++) {  
    TreeNode treeNode = new TreeNode(new String("Child " + "No." + i));  
    treeNode.setLevel(0);  
    root.addChild(treeNode);  
}
View treeView = new TreeView(root, context, new MyNodeViewFactory()).getView();
//add to view group where you want 
{% endhighlight %}

如果想了解更多细节或者改造使用可以在[GitHub][1]地址下载此项目，也欢迎提[Issue][2]。

[1]:	https://github.com/shinem/TreeView
[2]:	https://github.com/shinem/TreeView/issues