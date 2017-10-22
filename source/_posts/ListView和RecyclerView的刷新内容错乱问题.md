---
title: ListView和RecyclerView的刷新内容错乱问题
date: 2017-10-21 23:09:57
tags: Android
categories: Android
---

## 一、问题背景
ListView和RecyclerView是最常用的控件，但是因为itemView的复用，会导致往返滑动后，EditText编辑或者RadioButton等控件会出现显示不正常的问题。<!--more--> 这些问题都是因为第二次加载View的时候，缺少二次刷新后对控件的约束。因此解决问题的办法就是在初始化的时候添加约束。

## 二、问题展示

因为RecyclerView可以适配多种布局，功能比ListView强，因此以下例子都用RecyclerView示范。
### RadioButton的错乱。
布局很简单，List里面是单纯的RadioButton，Activity中单纯的就是RecyclerView，Adapter中，Creat和绑定Holder的代码如下：
```
@Override
 public TestHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        View view = LayoutInflater.from(parent.getContext()).inflate(R.layout.view_rec_item,parent,false);
        return new TestHolder(view);
    }

    @Override
    public void onBindViewHolder(TestHolder holder, int position) {
        String item = mlist.get(position);
        holder.testRbtn.setText(item);
    }
```
当点击5个RadioButton时：
![](http://wiki.hrsoft.net/uploads/201705/59258a9abf0e9_59258a9a.png)
下拉后会发现：
![](http://wiki.hrsoft.net/uploads/201705/59258af4d792e_59258af4.png)
然后再拉回：
![](http://wiki.hrsoft.net/uploads/201705/59258b032da49_59258b03.png)
会发现没有选择过的button被选中了，拉回后选中的却又没了。


## 三、如何解决：
前面提到过，所有的错乱原因都是itemView的复用造成，举个例子：对于RadioButton，当下拉时，因为前5个是点击的状态，复用View后，没有新的RadioButton的选择状态未被初始化，因此会错乱。

### 解决方法：
对于每个view，用一bool数组来记录当前所有RadioButton是否选中的状态，然后在初始化的时候通过判断数组中每一个元素的状态来设置RadioButton的状态。代码如下：
1.在构造函数中，创建flagList的实例，默认全部设置为false
```java
  public TestAdapter(List<String > list) {
        mlist = list;
        flagList = new ArrayList<>();
        for(int i =0 ;i<list.size();i++) {
            flagList.add(false);
        }
    }
```
2.在BindViewHolder的函数中，

```java

    @Override
    public void onBindViewHolder(TestHolder holder, final int position) {
        String item = mlist.get(position);
        holder.testRbtn.setText(item);
        if(flagList.get(position)) {
            holder.testRbtn.setChecked(true);
        } else {
            holder.testRbtn.setChecked(false);
        }
        holder.testRbtn.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                flagList.set(position,!flagList.get(position));
            }
        });
    }
```

这里对每个RadioButton做了初始化，判断flagList的每个元素的状态来设置RadioButton的状态，并且设置点击监听，点击后更新flagList。这样设置后，运行程序，会发现错乱的问题得到了解决。
**这里特别特别要注意一点：代码里面的if和else缺一不可，必须对每种状态都得初始化，如果仅有if没有else，还是会出现错乱的情况。**

## 四、总结

无论哪种的错位问题，都可以按照以上套路来。就是在加载每一项的View的时候将View中的控件初始化，比如EditText就添加TextChangedListener，文本改变后将文本存储在数组里面，然后加载View的时候取出，这样就可以避免错乱的情况了。