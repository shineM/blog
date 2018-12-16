---
layout: post
title:  Android属性动画实现shareElement动画平滑过渡
date:   2016-03-17
---

<p class="intro"><span class="dropcap">A</span>ndroid 5.0后加入了很多高级的Activity跳转效果，可以让App更加的Material Design，ActivityOptions自带的几种动画可以满足常用需求，比如makeSceneTransitionAnimation可以让两个页面的共享元素无缝衔接，适合一般图片跳转的使用场景，当然如果没有共享元素，我们也可以强行“共享”，你可以把一个floatActionButton和一整个activity页面设为共享，也能产生类似的缩放效果
</p>

这里就以fab按钮跳转为例，给startActivity加入参数就可以实现这种效果了，ActivityOptionsCompat 对于API 18也支持，而ActivityOptions只支持API 21以上，需要注意
{% highlight java %}
ActivityOptionsCompat options = ActivityOptionsCompat.makeSceneTransitionAnimation(MainActivity.this, fab, AddDiaryActivity.FAB_TRANSITION);  
startActivity(intent, options.toBundle());_
{% endhighlight %}

别忘了给fab和跳转页面设置相同的transitionName，不然没有效果，可以在xml文件指定，也可以在代码中通过setTransitionName设置，好了，上面短短几行代码就可以实现下面的效果了
<img src="{{ '/public/img/old.jpg' | prepend: site.baseurl }}" alt="">
仔细观察可以发现最后一点动画不是那么自然，突然就从白色小方块变成了绿色圆圈，为了更好的体验，我们需要让动画更平滑，于是我们需要从颜色和角的弧度两个属性进行变换。先不管怎么实现动画，我们最终都是调用

{% highlight java %}
activity.getWindow().setSharedElementReturnTransition(myTransition);//myTransition 就是需要实现的动画
{% endhighlight %}
所以就是一个自定义transition的问题，API 21中给出了几种transition的实现，ChangeBounds、ChangeTransform、ChangeClipBounds、ChangeImageTransform，适用场景最多的就是ChangeBounds，因为一般两个共享元素的layout大小位置都会产生变化，这里我们也以ChangeBounds为父类，自定义一个Transition，实现一个transition需要实现三个方法：捕捉开始和结束状态的属性captureStartValues，captureEndValues，实现动画createAnimator，我们这里需要的状态属性有两个：颜色和角的弧度，颜色可以用Integer，那么弧度呢？我们可以通过圆角的半径（float）来描述这个弧度大小。属性确定了，我们需要构建一个带有这两个属性的drawable对象，我们创建一个CornerDialog类来承载这两个属性
{% highlight java %}
public class CornerDialog extends Drawable {  

    public static final Property<CornerDialog, Integer> PROPERTY_COLOR = new Property<CornerDialog, Integer>(Integer.class, "color") {  
        @Override  
        public Integer get(CornerDialog object) {  
            return object.getColor();  
        }  
      
        @Override  
        public void set(CornerDialog object, Integer value) {  
            object.setColor(value);  
        }  
    };  
    public static final Property<CornerDialog, Float> PROPERTY_RADIUS = new Property<CornerDialog, Float>(Float.class, "radius") {  
        @Override  
        public Float get(CornerDialog object) {  
            return object.getRadius();  
        }  
      
        @Override  
        public void set(CornerDialog object, Float value) {  
            object.setRadius(value);  
        }  
    };  
    private int color;  
      
    private float radius;  
      
    private Paint paint;  
      
    public CornerDialog(int color, float radius) {  
        this.color = color;  
        this.radius = radius;  
        paint = new Paint(Paint.ANTI_ALIAS_FLAG);  
        paint.setColor(color);  
    }  
      
    @TargetApi(Build.VERSION_CODES.LOLLIPOP)  
    @Override  
    public void draw(Canvas canvas) {  
        //带圆角的矩形_

canvas.drawRoundRect(getBounds().left,getBounds().top,getBounds().right,getBounds().bottom,radius,radius,paint);  
    }  

    @Override  
    public void setAlpha(int alpha) {  
        paint.setAlpha(alpha);  
        invalidateSelf();  
    }  
      
    @Override  
    public void setColorFilter(ColorFilter colorFilter) {  
        paint.setColorFilter(colorFilter);  
        invalidateSelf();  
    }  
      
    @Override  
    public int getOpacity() {  
        return paint.getAlpha();  
    }  
      
    public float getRadius() {  
        return radius;  
    }  
      
    public void setRadius(float radius) {  
        this.radius = radius;  
    }  
      
    public int getColor() {  
        return color;  
    }  
      
    public void setColor(int color) {  
        this.color = color;  
        paint.setColor(color);  
    }  
}

{% endhighlight %}

然后在我们的自定义Transition类MyTransition中实现需要的三个方法
{% highlight java %}
@Override  
public void captureStartValues(TransitionValues transitionValues) {  
    super.captureStartValues(transitionValues);  
    transitionValues.values.put(PROPERTY_COLOR, Color.TRANSPARENT);  
    transitionValues.values.put(PROPERTY_RADIUS, 2.0f);  

}  

@Override  
public void captureEndValues(TransitionValues transitionValues) {  
    super.captureEndValues(transitionValues);  
    transitionValues.values.put(PROPERTY_COLOR, ContextCompat.getColor(transitionValues.view.getContext(), R.color.colorPrimary));  
    transitionValues.values.put(PROPERTY_RADIUS, (float) transitionValues.view.getWidth() / 2);  
}  

@TargetApi(Build.VERSION_CODES.LOLLIPOP)  
@Override  
public Animator createAnimator(ViewGroup sceneRoot, TransitionValues startValues, TransitionValues endValues) {  
    Animator changeBounds = super.createAnimator(sceneRoot, startValues, endValues);  
    if (startValues == null || endValues == null || changeBounds == null) {  
        return null;  
    }  
    Integer startColor = (Integer) startValues.values.get(PROPERTY_COLOR);  
    Float startRadius = (Float) startValues.values.get(PROPERTY_RADIUS);  
    Integer endColor = (Integer) endValues.values.get(PROPERTY_COLOR);  
    Float endRadius = (Float) endValues.values.get(PROPERTY_RADIUS);  
      
    CornerDialog drawable = new CornerDialog(startColor, startRadius);  
    startValues.view.setBackground(drawable);
//实现两个属性动画  
    Animator colorAnimator = ObjectAnimator.ofArgb(drawable, drawable.PROPERTY_COLOR, endColor);  
    Animator radiusAnimator = ObjectAnimator.ofFloat(drawable, drawable.PROPERTY_RADIUS, endRadius);  
    if (endValues.view instanceof ViewGroup) {  
        ViewGroup viewGroup = (ViewGroup) endValues.view;  
        for (int i = 0; i < viewGroup.getChildCount(); i++) {  
            View v = viewGroup.getChildAt(i);  
            v.animate().alpha(0f)  
                    .translationY(v.getHeight() / 3)  
                    .setDuration(50L)  
                    .setInterpolator(AnimationUtils.loadInterpolator(sceneRoot.getContext(), android.R.interpolator.fast_out_linear_in))  
                    .start();  
      
        }  
    }  


    AnimatorSet animatorSet = new AnimatorSet();  
    animatorSet.setDuration(300);  
    animatorSet.setInterpolator(AnimationUtils.loadInterpolator(sceneRoot.getContext(), android.R.interpolator.fast_out_slow_in));  
    animatorSet.playTogether(changeBounds, colorAnimator, radiusAnimator);  
    return animatorSet;  
}

{% endhighlight %}
最后在activity的oncreate中加入我们自定义的transition动画：
{% highlight java %}
activity.getWindow().setSharedElementReturnTransition(new 
MyTransition(color,radius));  
//这里的参数可以自己设置，也可以传入view参数，动态获取
{% endhighlight %}
下面是最终的效果图：
<img src="{{ '/public/img/smooth.jpg' | prepend: site.baseurl }}" alt="">
这里只实现了退出的动画，我们还可以再写一个transition类来实现进入的动画，把动画属性翻转过来就OK了，这个动画参考了开源项目Plaid，非常感谢作者，从其中学到了很多，传送门[GitHub][1]

[1]:	https://github.com/nickbutcher/plaid