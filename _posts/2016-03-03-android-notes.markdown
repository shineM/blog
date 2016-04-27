---
layout: post
title:  近期Android学习笔记
date:   2016-03-03
---

<p class="intro"><span class="dropcap">好</span>久没更新博客了，写博客确实能把学到的知识更系统地消化一遍，但是实在是耗时间。最近Android学习进入了痴迷的状态，下班时间和周末基本泡在了Android Studio里面，成就感可能就是动力的来源吧，经常为解决了一个小bug、实现了一个小动画而激动一小会儿。嗯，学习贵在坚持！</p>

<p>这里记录一下近期的Android学习笔记，大多数来源于平时记录在印象笔记里面的碎片，可能太基础了，算是温习一遍吧</p>

#### RecylerView上拉加载更多数据的实现
写一个继承自RecyclerView.OnScrollListener滑动监听
{% highlight java %}
public abstract class InfiniteScrollListener extends RecyclerView.OnScrollListener
{% endhighlight %}
重写onScroll方法，当滑动距离超过一定Item数目的时候就触发onLoadMore的抽象方法
{% highlight java %}
@Override  
public void onScrolled(RecyclerView recyclerView, int dx, int dy) {  
    if (dy \< 0 || dataLoading.isDataLoading()) return;  
  
    final int visibleItemCount = recyclerView.getChildCount();  
    final int totalItemCount = layoutManager.getItemCount();  
    final int firstVisibleItem = layoutManager.findFirstVisibleItemPosition();  
  
    if ((totalItemCount - visibleItemCount) \<= (firstVisibleItem + VISIBLE_THRESHOLD)) {  
        onLoadMore();  
    }  
}
{% endhighlight %}
最后在外部添加该滑动监听事件，并实现方法onLoadMore
{% highlight java %}
grid.addOnScrollListener(new InfiniteScrollListener(layoutManager, dataManager) {  
    @Override  
    public void onLoadMore() {  
        dataManager.loadAllDataSources();  
    }  
});

{% endhighlight %}
#### Picasso加载本地图片的问题
路径前面要加“file：”
{% highlight java %}
Picasso.with(this).load("file:"+imagePath).fit().into(imageView);
{% endhighlight %}

#### 输入键盘和View的布局适应问题
当输入框下面的view没有被输入法遮挡，而是推上来导致view变形的时候，可以在manifest中设置该Activity的属性为：
{% highlight java %}
android:windowSoftInputMode="stateHidden|adjustNothing"
{% endhighlight %}

#### Java String split方法
为了把一个字符串数组保存到数据库，我把它们拼接成一个字符串，中间以符号“|”连接，然后读取的时候采用split方法拆分，然后就会出问题。
我们看jdk doc中说明  
{% highlight java %}

public String[]() split(String regex)
 Splits this string around matches of the given regular expression.
{% endhighlight %}

参数regex是一个 regular-expression的匹配模式而不是一个简单的String，他对一些特殊的字符可能会出现你预想不到的结果，因此得采用
{% highlight java %}
String[]() aa = "aaa|bbb|ccc".split(“\\|”);
{% endhighlight %}
#### 从sd卡加载图片时，显示为倒立的解决办法
可以调用exif.getAttributeInt方法获取图片的翻转属性，根据返回的角度进行校正，这里展示了一个读取本地图片进行剪裁（calculateInSampleSize方法参考官方文档）和翻转校正的方法
{% highlight java %}
public static Bitmap decodeSampledBitmapFromResource(String filePath, int reqWidth, int reqHeight) {  
  
    // First decode with inJustDecodeBounds=true to check dimensions  
    final BitmapFactory.Options options = new BitmapFactory.Options();  
    options.inJustDecodeBounds = true;  
    BitmapFactory.decodeFile(filePath, options);  
  
    // Calculate inSampleSize  
    options.inSampleSize = calculateInSampleSize(options, reqWidth, reqHeight);  
  
    // Decode bitmap with inSampleSize set  
    options.inJustDecodeBounds = false;  
    Bitmap bitmap = BitmapFactory.decodeFile(filePath, options);  
    try {  
        ExifInterface exif = new ExifInterface(filePath);  
        int orientation = exif.getAttributeInt(ExifInterface.TAG_ORIENTATION, 1);  
        Log.d("EXIF", "Exif: " + orientation);  
        Matrix matrix = new Matrix();  
        if (orientation == 6) {  
            matrix.postRotate(90);  
        } else if (orientation == 3) {  
            matrix.postRotate(180);  
        } else if (orientation == 8) {  
            matrix.postRotate(270);  
        }  
        bitmap = Bitmap.createBitmap(bitmap, 0, 0, bitmap.getWidth(), bitmap.getHeight(), matrix, true); // rotating bitmap  
  
    } catch (Exception e) {  
  
    }  
    return bitmap;  
}

{% endhighlight %}
#### 裁取图片的中间部分，返回一个宽度为原始宽高较小值的正方形
{% highlight java %}
public static Bitmap centerSquareScaleBitmap(Bitmap bitmap, int edgeLength) {  
    if (null == bitmap || edgeLength \<= 0) {  
        return null;  
    }  
  
    Bitmap result = bitmap;  
    int widthOrg = bitmap.getWidth();  
    int heightOrg = bitmap.getHeight();  
  
    if (widthOrg \> edgeLength && heightOrg \> edgeLength) {  
        //压缩到一个最小长度是edgeLength的bitmap  
        int longerEdge = (int) (edgeLength * Math.max(widthOrg, heightOrg) / Math.min(widthOrg, heightOrg));  
        int scaledWidth = widthOrg \> heightOrg ? longerEdge : edgeLength;  
        int scaledHeight = widthOrg \> heightOrg ? edgeLength : longerEdge;  
        Bitmap scaledBitmap;  
  
        try {  
            scaledBitmap = Bitmap.createScaledBitmap(bitmap, scaledWidth, scaledHeight, true);  
        } catch (Exception e) {  
            return null;  
        }  
  
        //从图中截取正中间的正方形部分。  
        int xTopLeft = (scaledWidth - edgeLength) / 2;  
        int yTopLeft = (scaledHeight - edgeLength) / 2;  
  
        try {  
            result = Bitmap.createBitmap(scaledBitmap, xTopLeft, yTopLeft, edgeLength, edgeLength);  
            scaledBitmap.recycle();  
        } catch (Exception e) {  
            return null;  
        }  
    }  
  
    return result;  
}

{% endhighlight %}

#### 字符串小技巧
 — 判断是否包含某个字符
{% highlight java %}
if (-1 == name.indexOf('.'))
{% endhighlight %}
 — 判断字符串是否为空
android 判断editText是否为空，推荐使用TextUtil.isEmpty（）
比如使用editText.getText().toString()=“”会把hint元素算进来
#### Android M权限问题
Android M开始对权限管理更严格了，读取和写入操作权限除了在manifest中配置外还需要单独申请，这里以读取本地图片为例，在执行读取操作前，需要检查权限，如果没有，就弹出窗口向用户申请
{% highlight java %}
private void checkPermission() {  
    int storagePermission = ContextCompat.checkSelfPermission(this, Manifest.permission.WRITE_EXTERNAL_STORAGE);  
    String[]() reqPermissonList = {Manifest.permission.WRITE_EXTERNAL_STORAGE};  
  
    if (storagePermission != PackageManager.PERMISSION_GRANTED) {  
        ActivityCompat.requestPermissions(this, reqPermissonList, REQUEST_WRITE_PERMISSION_CODE);  
    } else {  
        startChooseDialog();  
    }  
}

{% endhighlight %}
这里接收用户是否授予权限，然后根据返回码执行后面的选择操作
{% highlight java %}
@Override  
public void onRequestPermissionsResult(int requestCode, String[]() permissions, int[]() grantResults) {  
    switch (requestCode) {  
        case REQUEST_WRITE_PERMISSION_CODE: {  
            for (int i = 0; i \< permissions.length; i++) {  
                if (grantResults[i]() == PackageManager.PERMISSION_GRANTED) {  
                    startCameraIntent();  
                    Log.d("Permissions", "Permission Granted: " + permissions[i]());  
                } else if (grantResults[i]() == PackageManager.PERMISSION_DENIED) {  
                    Log.d("Permissions", "Permission Denied: " + permissions[i]());  
                }  
            }  
        }  
        break;  
        default: {  
            super.onRequestPermissionsResult(requestCode, permissions, grantResults);  
        }  
    }  
}

{% endhighlight %}
#### 防止多次点击事件
有时候的连续点击一个view会触发多个事件，比如Activity跳转，为避免这种情况发生可以在执行点击事件之前判断一下前后点击的时间差
{% highlight java %}
private static long lastClickTime;  
  
public static boolean isFastDoubleClick() {  
    long time = System.currentTimeMillis();  
    long timeD = time - lastClickTime;  
    if (0 \< timeD && timeD \< 800) {  
        return true;  
    }  
    lastClickTime = time;  
    return false;  
}

{% endhighlight %}
#### 录制GIF
这种方法录制GIF分为两步完成：录视频，剪裁成GIF。
首先手机连上电脑，确认adb命令可以使用，然后在terminal输入以下命令
{% highlight java %}
adb shell screenrecord /sdcard/xxx.mp4
{% endhighlight %}

就会录制一段视频保存在指定位置了，还可以加入一些录制参数
{% highlight java %}
 --time-limit N //限制视频录制时间为N秒,默认180秒
 --size N*//限制录制视频分辨率为N*N，默认使用手机的分辨率
--bit-rate //指定视频的比特率为6Mbps，默认为4Mbps
{% endhighlight %}
接着需要用到一款软件Video To GIF ，下载地址[豌豆荚][10]。


[10]:	http://apps.wandoujia.com/redirect?signature=e3f9891&url=http%3A%2F%2Fshouji.360tpcdn.com%2F140404%2Fab6403ea7dfea7715265e25f1c19cff1%2Fnet.atredroid.videotogif_23.apk&pn=net.atredroid.videotogif&md5=ab6403ea7dfea7715265e25f1c19cff1&apkid=10172992&vc=23&size=9947938&pos=t/detail&tokenId=sspai&appType=APP "豌豆荚"