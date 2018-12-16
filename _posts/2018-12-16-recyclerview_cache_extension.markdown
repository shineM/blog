---
layout: post
title:  列表优化另辟蹊径
date:   2018-12-16
---

 **FBI Warning**: 本文的方案有和 RecyclerView 设计背道而驰的意味，未经线上实测，小心使用，首先声明一下本方案的限制：

- onCreateViewHolder 和 onBindViewHolder 不能有只允许在主线程执行的操作，如：DataBinding
- 会牺牲一点额外的内存空间，比 RecyclerView 默认缓存会大一点
- 对 onBindViewHolder 的优化只能向下滑（正常浏览 feed 的方向）

带来的好处：

- 首屏渲染开销大幅减小

- 滑动流畅。你甚至可以在 onBindViewHolder 来几个 Thread.sleep() 之类的操作

  ​

####  列表常规优化思路
**Profiler 分析耗时操作**

比起打 Log 看耗时 Profiler 更加直接，能快速抓到卡顿元凶。一般来说，如果列表性能不好，都是 onCreateViewHolder 和 onBindViewHolder 不够轻量造成的，或者在 RecyclerView 的某些回调（比如 onScrollChanged 等）中有耗时操作，找到它然后怎么解决就看具体情况而定了。

**解决首屏卡顿**

我们知道 RecyclerView 的 ViewHolder 创建一般发生在第一次更新数据时，因为开始没有可以复用的 ViewHolder，通过 Profiler 发现，大部分 CPU 计算都消耗在了 inflate 布局上，当列表布局非常复杂的时候更为明显，那有办法让 View 的创建更快吗？没有，除非优化布局，但布局的复杂度和业务有很大关系，对于首页是列表的应用来说，启动阶段要做的事太多了，所以我们最好还是要避免过多的 inflate。其中一个解决办法是在加载数据的阶段（子线程）去创建 View，只要 View 不被 attach 到 ViewRoot 上面，new View() 的操作是可以放在子线程的，同理 LayoutInflater 也是。当然这样做可能会有些丑陋，我们需要先用数据去拿到对应的 ViewHolder 类型，然后去创建对应的布局，当列表数据类型非常多，且业务经常变动的时候，我们很难去维护一个庞大的 Data - ViewHolder - layout.xml 映射关系。要不直接缓存 ViewHolder 好了？甚至能不能实现提前执行 bind？ 到这里我算是彻底走火入魔了。。

#### 从源码找黑魔法入口

**（不想看源码的可以直接到最后一步）**

以 LinearLayoutManager 为例，我们看下 ViewHolder 的创建和复用流程。

*LinearLayoutManager.java*

`

	int scrollBy(int dy, Recycler recycler, State state)  {
	    //...
	    // 1）更新滑动后可用高度
	    this.updateLayoutState(layoutDirection, absDy, true, state)
	    // 2) 如果有可用高度则填充直到超出			
	    int consumed = this.mLayoutState.mScrollingOffset + this.fill(recycler, this.mLayoutState, state, false);
	    //...
	
	}
`

无论是首屏渲染还是滑动更新，最终都是走 updateLayoutState->fill 的流程，先看 *updateLayoutState*

`

	private void updateLayoutState(int layoutDirection, int requiredSpace, boolean canUseExistingSpace, State state) {
	    //...
	    int scrollingOffset;
	    View child;
	    if (layoutDirection == 1) {
	    	//...
	    	child = this.getChildClosestToEnd();
	    	// 算出最后一个 Item 超出屏幕多长。（最后一个 Item 的 bottom - LayoutManager 的可见 bottom）		            
	        scrollingOffset = this.mOrientationHelper.getDecoratedEnd(child) - this.mOrientationHelper.getEndAfterPadding();
	    } else {
	        //...
	    }
	    this.mLayoutState.mAvailable = requiredSpace;
	    if (canUseExistingSpace) {
	    	// 滑动之后可用的高度。比如原来的 scrollingOffset 是 100，但是只滑动了 20，mAvailable 就是 -80，最后一条还是有 80 的高度处于底部不可见区域，则后面的填充逻辑不会执行。
	        this.mLayoutState.mAvailable -= scrollingOffset;
	    }
	    this.mLayoutState.mScrollingOffset = scrollingOffset;
	
	}		
`

*updateLayoutState* 简单来说就是算下可用空间还有多少，算完后交给 *fill* 去填充，看下 *fill* 的具体过程

`

	int fill(Recycler recycler, LinearLayoutManager.LayoutState layoutState, State state, boolean stopOnFocusable) {
	    int start = layoutState.mAvailable;
	   	//...
	    int remainingSpace = layoutState.mAvailable + layoutState.mExtra;
	    LinearLayoutManager.LayoutChunkResult layoutChunkResult = this.mLayoutChunkResult;
	    // 一直填充直到 remainingSpace < 0 或者全部数据都已经填充
	    while((layoutState.mInfinite || remainingSpace > 0) && layoutState.hasMore(state)) {
	        //...
	        // 填充一个 ViewHolder
	        this.layoutChunk(recycler, state, layoutState, layoutChunkResult);
	        //...
	        layoutState.mOffset += layoutChunkResult.mConsumed * layoutState.mLayoutDirection;
	        if (!layoutChunkResult.mIgnoreConsumed || this.mLayoutState.mScrapList != null || !state.isPreLayout()) {
	            layoutState.mAvailable -= layoutChunkResult.mConsumed;
	            // 减去刚刚那个 ViewHolder 占据掉的空间，继续循环
	            remainingSpace -= layoutChunkResult.mConsumed;
	        }
	        //...
	    }
	    return start - layoutState.mAvailable;
	
	 }
`

fill 就和方法名一样，填充 layout 直到占满为止，到这里其实和 ViewHolder 的创建回收复用关系都不大，继续跟进 layoutChunk 看

`

	void layoutChunk(Recycler recycler, State state, LinearLayoutManager.LayoutState layoutState, LinearLayoutManager.LayoutChunkResult result) {
 		// 去 Recycler 中取 View
        View view = layoutState.next(recycler);
        //...
        LayoutParams params = (LayoutParams)view.getLayoutParams();
        if (this.mShouldReverseLayout == (layoutState.mLayoutDirection == -1)) {
       		// 加到 RecyclerView 里面
            this.addView(view);
        } else {
            this.addView(view, 0);
        }
        //...
        // 测量出 ViewHolder 初始大小
        this.measureChildWithMargins(view, 0, 0);
        result.mConsumed = this.mOrientationHelper.getDecoratedMeasurement(view);
        //...
        this.layoutDecoratedWithMargins(view, left, top, right, bottom);
       	//...
    }
`

*layoutChunk* 就是填充单个 ViewHolder 的过程，这里我们已经到了获取 ViewHolder 的入口了，下面测量宽高和 decorate 的步骤我们省领掉，直接进入正题 *next* 方法

LayoutState.class
`

	View next(Recycler recycler) {
	    if (this.mScrapList != null) {
	        return this.nextViewFromScrapList();
	    } else {
	        View view = recycler.getViewForPosition(this.mCurrentPosition);
	        this.mCurrentPosition += this.mItemDirection;
	        return view;
	    }
	
	}

`

进入 Recycler

`

	RecyclerView.ViewHolder tryGetViewHolderForPositionByDeadline(int position, boolean dryRun, long deadlineNs) {
	    if (position >= 0 && position < RecyclerView.this.mState.getItemCount()) {
	        boolean fromScrapOrHiddenOrCache = false;
	        RecyclerView.ViewHolder holder = null;
	        // 0) If there is a changed scrap, try to find from there
	    	if (mState.isPreLayout()) {
	      	  	holder = getChangedScrapViewForPosition(position);
	       	 	fromScrapOrHiddenOrCache = holder != null;
	  		}
	        // 1) Find by position from scrap/hidden list/cache
	        if (holder == null) {
	            holder = getScrapOrHiddenOrCachedHolderForPosition(position, dryRun);
	            //...
	        }
	        final int type = mAdapter.getItemViewType(offsetPosition);
	        // 2) Find from scrap/cache via stable ids, if exists
	        if (mAdapter.hasStableIds()) {
	            holder = getScrapOrCachedViewForId(mAdapter.getItemId(offsetPosition),
	                    type, dryRun);
	           //...
	        }
	        // 3) 从自定义缓存 mViewCacheExtension 获取
	        if (holder == null && mViewCacheExtension != null) {
	            final View view = mViewCacheExtension
	                    .getViewForPositionAndType(this, position, type);
	            //...
	        }
	        // 4) 从 RecycledViewPool 获取
	        if (holder == null) { // fallback to pool
	            //...
	            holder = getRecycledViewPool().getRecycledView(type);
	        }
	        // 5) 没有缓存，创建
	        if (holder == null) {
	            holder = mAdapter.createViewHolder(RecyclerView.this, type);
	            //...
	        }
	        
	        if (!holder.isBound() || holder.needsUpdate() || holder.isInvalid()) {
	                type = RecyclerView.this.mAdapterHelper.findPositionOffset(position);
	                bound = this.tryBindViewHolderByDeadline(holder, type, position, deadlineNs);
	            }
	        }
	        //..
	        return holder;
	    }
	}
`

这个方法就是创建和复用的核心部分了，第一步从 scrap 或者 cache 缓存获取，scrap 缓存是还 attach 在 RecyclerView 上的 ViewHolder，滚动列表的时候会把当前所有可见的 ViewHolder 添加到 scrap 里面，紧接着会被拿出来直接用，而 cache 是一个默认大小为 2 的缓存空间，当有 ViewHolder 被滑出屏幕区域外一定距离，被被 recycle 然后加到 cache 里面，如果 cache 数量到达最大则会丢进 RecyclerPool；第二步先省略，我们一般不会设置 stableId；第三步，从自定义缓存获取，好了，终于到了我们想要的了，官方文档解释是让开发者自己决定缓存的策略，但我很少看到有应用的场景，那我们下面就开始试一下是否能实现我们激进的优化方案。 



#### 最终实现

给 RecycleView 设置 ViewCacheExtension

`mRecyclerView.setViewCacheExtension(mExtensionCache);`

请求完数据的时候开始缓存 Holder

`

	 private void loadMoreData() {
	    mIsLoading = true;
	    final int currentSize = mDataList.size();
	    mDataList.add(LOADING_HOLDER);
	    mAdapter.notifyItemInserted(mDataList.size());
	    new Thread(new Runnable() {
	        @Override
	        public void run() {
	            try {
	                Thread.sleep(500);
	            } catch (InterruptedException e) {
	                e.printStackTrace();
	            }
	            final List<Object> newDataList = new ArrayList<>();
	            for (int i = 0; i < 10; i++) {
	                newDataList.add(currentSize + i);
	            }
	            // 开始缓存
	            mExtensionCache.cacheHoldersForDataList(mRecyclerView, newDataList);
	
	            runOnUiThread(new Runnable() {
	                @Override
	                public void run() {
	                    mDataList.remove(LOADING_HOLDER);
	                    mDataList.addAll(newDataList);
	                    mAdapter.notifyDataSetChanged();
	                    mIsLoading = false;
	                }
	            });
	        }
	    }).start();
	}
`

`

```
public void cacheHoldersForDataList(RecyclerView recyclerView, List<Object> dataList) {
    mDataListForBind.clear();
    mDataListForBind.addAll(dataList);
    mCacheHolders.clear();
    for (Object data : dataList) {
        int position = dataList.indexOf(data);
        // mMirrorAdapter 是原始 Adapter 的另外一个实例，避免和原来的 Adapter 数据混淆在一起
        RecyclerView.ViewHolder holder = mMirrorAdapter.createViewHolder(
                recyclerView, mMirrorAdapter.getItemViewType(position));

        // 这一步模拟正常的创建过程
        RecyclerView.LayoutParams params = (RecyclerView.LayoutParams) holder.itemView.getLayoutParams();
        try {
            Field field = RecyclerView.LayoutParams.class.getDeclaredField("mViewHolder");
            field.setAccessible(true);
            field.set(params, holder);
            holder.itemView.setLayoutParams(params);

        } catch (NoSuchFieldException | IllegalAccessException e) {
            e.printStackTrace();
        }
		// 开始 bind，如果你只想要优化首屏的速度，这一步注释掉就好了
        mMirrorAdapter.bindViewHolder(holder, position);

        // 虽然我们已经执行过 bind 了但是 ViewHolder 里面的状态、位置等信息都和实际的是不一样的，
        // 我么需要让即将到来的主线程 onBind 再执行一次，让 RecyclerView 把 ViewHolder 的信息置为
        // 正确的状态，1<<2 就是 ViewHolder.FLAG_INVALID 的值
        try {
            Method method = RecyclerView.ViewHolder.class.getDeclaredMethod("addFlags", int.class);
            method.setAccessible(true);
            method.invoke(holder, 1 << 2);
        } catch (NoSuchMethodException | IllegalAccessException | InvocationTargetException e) {
            e.printStackTrace();
        }
		// 加入自定义缓存
        mCacheHolders.put(data, holder);
    }
}
```

`

接着就是让 Recycler 走到我们的自定义缓存逻辑了，这里的 position 不要直接用 mDataListForBind 去取，我们需要从原始的列表去拿正确的数据

`

```
@Nullable
@Override
public View getViewForPositionAndType(@NonNull RecyclerView.Recycler recycler, int position, int viewType) {
    RecyclerView.ViewHolder cacheHolder = getCacheHolder(mDataProvider.getDataAt(position));
    return cacheHolder != null ? cacheHolder.itemView : null;
}

```

`

具体代码和实现效果可以在[这里][1]查看。

至此一个简单的 ViewCacheExtension 就完成了，此方案仅提供一个思路，不建议直接用到线上项目，列表优化的最佳实践依然是老老实实减少主线程的耗时操作。

#### 

[1]:	https://github.com/shineM/RecyclerViewCacheDemo