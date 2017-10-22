---
title: 利用RecyclerView实现简单的布局混搭
date: 2017-10-21 23:06:57
tags: Android
categories: Android
---

## 一、Header和Footer

有时候我们需要在ListView或者RecyclerView中添加一个Header或者一个Footer，ListView中有添加Header和Footer的函数，而RecyclerView中并没有。所以需要简单的配置来实现RecyclerView的Header和Footer。实现过程很简单，只需要重写自带的getItemViewType和onCreateViewHolder函数，然后自定义不同的holder，在bindViewHolder根据不同类型的holder来进行配置。<!--more-->


### 首先，不要让adapter继承自自定义的Holder


```java

public class MyAdapter extends RecyclerView.Adapter < RecyclerView.ViewHolder >

```



### 其次，定义变量，重写getItemViewType方法和getCount的方法


```java
/item类型 
    public static final int ITEM_TYPE_HEADER = 0;
    public static final int ITEM_TYPE_CONTENT = 1;
    public static final int ITEM_TYPE_BOTTOM = 2;

    private int mHeaderCount=1;//头部View个数
    private int mBottomCount=1;//底部View个数
 @Override
    public int getItemViewType(int position) {
        int dataItemCount = getContentItemCount();
        if (mHeaderCount != 0 && position < mHeaderCount) {
            return ITEM_TYPE_HEADER;
        } else if (mBottomCount != 0 && position >= (mHeaderCount + dataItemCount)) {
            return ITEM_TYPE_BOTTOM;
        } else {
            return ITEM_TYPE_CONTENT;
        }
    }
	 @Override
	public int getItemCount() {}
       return mHeaderCount + getContentItemCount() + mBottomCount;
    }
```


### 然后根据不同类型的type返回不同的View，并且设置不同的ViewHolder


```java
//内容的Holder
public static class ContentViewHolder extends RecyclerView.ViewHolder {
        private TextView textView;
        public ContentViewHolder(View itemView) {
            super(itemView);
        }
    }
    //头部Holder
    public static class HeaderViewHolder extends RecyclerView.ViewHolder {

        public HeaderViewHolder(View itemView) {
            super(itemView);
        }
    }
    //底部Holder
    public static class BottomViewHolder extends RecyclerView.ViewHolder {
        public BottomViewHolder(View itemView) {
            super(itemView);
        }
    }
//根据不同的type获取不同的ViewHolder
 @Override
    public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        if (viewType ==ITEM_TYPE_HEADER) {
            return new HeaderViewHolder(mLayoutInflater.inflate(R.layout.rv_header, parent, false));
        } else if (viewType == mHeaderCount) {
            return  new ContentViewHolder(mLayoutInflater.inflate(R.layout.rv_item, parent, false));
        } else if (viewType == ITEM_TYPE_BOTTOM) {
            return new BottomViewHolder(mLayoutInflater.inflate(R.layout.rv_footer, parent, false));
        }
        return null;
    }
	
	
```


### 这样添加Header和Footer的功能就简单的实现完成了


## 二、GridLayoutManager来实现简单的混搭布局


其实思想和上面的差不多，重写itemViewType，然后根据不同的type来获取不同的View。但是当List为多列的瀑布流的时候，就需要用到GridLayoutManager来管理了，比如B站的布局。此时需要根据当前item类型来判断占据的横向格数。adapter里面的代码和上面类似就不贴了，现在需要改变的是占据的格数，代码如下：
```java
gridManager.setSpanSizeLookup(new GridLayoutManager.SpanSizeLookup() {
            @Override
            public int getSpanSize(int position) {
                if(gradAdapter.getItemViewType(position) == MyAdapter.LABEL_VIEW_TYPE) {
                    return 2;
                }
                return 1;
            }
        });
```
其中gridManager创建时，定义其spanCount为2，当当前的ViewType为LABEL_VIEW_TYPE时，沾满横向的格数，否则占一个。这里可以根据实际情况来设置不同的spanCount，然后根据不同的ViewType来设置不同的格数，以实现简单的布局混搭。效果图如下
![](http://wiki.hrsoft.net/uploads/201705/592d64eda10d0_592d64ed.png)
