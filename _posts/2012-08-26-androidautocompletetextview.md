---
layout: post
title: "Android浏览器地址栏中历史访问记录的自动提示实现(AutoCompleteTextView)"
description: ""
category: 
tags: []
---
{% include JB/setup %}

目前无论是PC上还是手机上，浏览器的地址栏都带有历史访问记录自动提示功能，例如之前访问过[http://www.qq.com](http://www.qq.com)，那么当下次再次输入qq，或者www.q的时候（具体出发规则可定制），[http://www.qq.com](http://www.qq.com)就会在地址栏下面以下拉窗口的形式自动给出提示，方便用户选择并完成地址的输入工作。

在Android中，通过sdk提供的`AutoCompleteTextView`我们可以完成类似的功能，当然这里布局限于浏览器的地址提示，任何历史记录都可以通过该控件来实现，下面将通过代码来说明具体的实现过程。

要实现历史记录的提示功能，首先要解决的问题就是历史记录的存储，Android中提供了几种存储方式，官方sdk文档中给出了详细的说明：[http://developer.android.com/guide/topics/data/data-storage.html](http://developer.android.com/guide/topics/data/data-storage.html) ，大概分析了一下，对于浏览器历史记录而言，SQLite应该是最合适的存储方式，Android原生浏览器也使用了SQLite的存储方式，因此本文也基于SQLite来保存历史记录。对于SQLite的介绍可以参考这篇文章：[http://www.ibm.com/developerworks/cn/opensource/os-cn-sqlite/](http://developer.android.com/guide/topics/data/data-storage.html)，下面就先来实现历史记录的存储部分。

Android 提供了 `SQLiteOpenHelper` 来创建数据库，我们只要继承 `SQLiteOpenHelper` 类，就可以轻松的创建数据库，并在`onCreate()`中创建需要用到的表，代码如下：

{% highlight java %}
import android.content.Context;
import android.database.sqlite.SQLiteDatabase;
import android.database.sqlite.SQLiteOpenHelper;

public class SuggestionDBHelper extends SQLiteOpenHelper{

    private static final String DBNAME = "url.db";// 数据库名
    
    public SuggestionDBHelper(Context context){
        super(context, DBNAME, null, DATABASE_VERSION);
    }

    @Override
    public void onCreate(SQLiteDatabase db) {
        db.execSQL("CREATE TABLE IF NOT EXISTS history" +  
                "(_id INTEGER PRIMARY KEY AUTOINCREMENT, url TEXT, title TEXT)");//创建history表，包含id，url和title字段
    }

    @Override
    public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {}

}
{% endhighlight %}

为了解除上层逻辑和数据库操作之间的耦合，我们通过创建类`SuggestionDBManage`来封装url历史记录的添加和删除操作，这里主要实现了历史记录的`insert`和`query`函数，代码如下，代码不难理解：

{% highlight java %}
import java.util.ArrayList;
import java.util.List;

import android.content.Context;
import android.database.Cursor;
import android.database.sqlite.SQLiteDatabase;

import com.tencent.test.SuggestionItem;

public class SuggestionDBManage {
    private SuggestionDBHelper mDateBaseHelper;
    private SQLiteDatabase mDatabase;

    public SuggestionDBManage(Context context) {
        mDateBaseHelper = new SuggestionDBHelper(context);
        mDatabase = mDateBaseHelper.getWritableDatabase();
    }

    public void insert(SuggestionItem urlitem) {
        mDatabase.beginTransaction(); // 开始事务
        try {
            mDatabase.execSQL("INSERT INTO history VALUES(null, ?, ?)",
                    new Object[] { urlitem.getUrl(), urlitem.getTitle() });
            mDatabase.setTransactionSuccessful(); // 设置事务成功完成
        } finally {
            mDatabase.endTransaction(); // 结束事务
        }
    }

    public List<SuggestionItem> query(String prefix) {
        String like = prefix + "%";
        List<SuggestionItem> values = new ArrayList<SuggestionItem>();
        final String selection = "(url LIKE ? OR url LIKE ? OR url LIKE ? OR url LIKE ?)";
        String[] selectionArgs = new String[4];
        selectionArgs[0] = "http://" + like;
        selectionArgs[1] = "http://www." + like;
        selectionArgs[2] = "https://" + like;
        selectionArgs[3] = "https://www." + like;
        Cursor cursor = mDatabase.query("history", new String[] { "url", "title" }, selection,
                selectionArgs, null, null, null);
        cursor.moveToFirst();
        for (int i = 0; i < cursor.getCount(); i++) {
            String url = cursor.getString(0);
            String title = cursor.getString(1);
            values.add(new SuggestionItem(url, title));
            cursor.moveToNext();
        }
        return values;
    }
    
    public void close(){
        mDatabase.close();
    }
}
{% endhighlight %}

为了实现历史记录的自动提示功能，我们需要借助Android提供的`AutoCompleteTextView`控件，该控件使用了`Adapter`方式来绑定用来提示的数据，用过类似ListView控件的话应该对这种方式不会陌生，然而对于`AutoCompleteTextView`来说，绑定的Adapter必须是实现了`Filterable`接口的`Adapter`，关键字的过滤功能是通过`Filter`这个类来实现的，而`AutoCompleteTextView`会调用`Filterable`接口的`getFilter()`函数，得到具体的`Filter`实例后执行过滤，具体的过滤规则是通过继承`Filter`这个类，并且重载`Filter`类的`performFiltering`函数来定制的。
数据库的访问通过`AsnycTask`来完成，这是考虑到数据库的`query`操作有可能是一个比较耗时的操作，因此不适合放到主线程中，通过`AsycTaskv`可以避免界面失去响应的问题，从数据库取得数据后通过`notifyDataSetChanged()`来更新界面，具体代码如下：

{% highlight java %}
import java.util.ArrayList;
import java.util.List;

import android.content.Context;
import android.os.AsyncTask;
import android.view.LayoutInflater;
import android.view.View;
import android.view.ViewGroup;
import android.widget.BaseAdapter;
import android.widget.Filter;
import android.widget.Filterable;
import android.widget.TextView;

import com.tencent.urldatabase.SuggestionDBManage;

public class SuggestionAdapter extends BaseAdapter implements Filterable {

    private Context context;
    private SuggestionDBManage mDbManage;
    private ArrayFilter mFilter;
    private List<SuggestionItem> mFilterItems = new ArrayList<SuggestionItem>();// 过滤后的item

    public SuggestionAdapter(Context context, SuggestionDBManage dbManage) {
        this.context = context;
        mDbManage = dbManage;
    }

    @Override
    public Filter getFilter() {
        if (mFilter == null) {
            mFilter = new ArrayFilter();
        }
        return mFilter;
    }

    private class SuggestionAsyncTask extends AsyncTask<String, Void, List<SuggestionItem>>{

        @Override
        protected List<SuggestionItem> doInBackground(String... params) {
            List<SuggestionItem> suggestionItems = mDbManage.query(params[0]);
            return suggestionItems;
        }

        @Override
        protected void onPostExecute(List<SuggestionItem> result) {
            mFilterItems = result;
            notifyDataSetChanged();
            super.onPostExecute(result);
        }        
        
    }
    
    private class ArrayFilter extends Filter {

        @Override
        protected FilterResults performFiltering(CharSequence prefix) {
            String prefixString = prefix.toString().toLowerCase();
            startSuggestionsAsync(prefixString);
            return null;
        }

        private void startSuggestionsAsync(String prefixString) {
            new SuggestionAsyncTask().execute(prefixString.toString());
        }

        @Override
        protected void publishResults(CharSequence constraint,
                FilterResults results) {}

    }

    @Override
    public int getCount() {
        return mFilterItems.size();
    }

    @Override
    public Object getItem(int position) {
        return mFilterItems.get(position);
    }

    @Override
    public long getItemId(int position) {
        return position;
    }

    @Override
    public View getView(final int position, View convertView, ViewGroup parent) {
        ViewHolder holder = null;
        if (convertView == null) {
            holder = new ViewHolder();
            LayoutInflater inflater = (LayoutInflater) context
                    .getSystemService(Context.LAYOUT_INFLATER_SERVICE);
            convertView = inflater.inflate(
                    R.layout.simple_list_item_for_autocomplete, null);
            holder.url = (TextView) convertView.findViewById(R.id.url);
            holder.title = (TextView) convertView.findViewById(R.id.title);
            convertView.setTag(holder);
        } else {
            holder = (ViewHolder) convertView.getTag();
        }

        holder.url.setText(mFilterItems.get(position).getUrl());
        holder.title.setText(mFilterItems.get(position).getTitle());
        return convertView;
    }

    class ViewHolder {
        TextView url;
        TextView title;
    }
}
{% endhighlight %}

SuggestionItem.java:

{% highlight java %}
public class SuggestionItem {
    private String mUrl;
    private String mTitle;
    
    public SuggestionItem(String url, String title) {
        mUrl = url;
        mTitle = title;
    }

    public String getUrl() {
        return mUrl;
    }

    public String getTitle() {
        return mTitle;
    }
}
{% endhighlight %}

MainActivity.java

{% highlight java %}
import android.app.Activity;
import android.os.Bundle;
import android.view.View;
import android.view.View.OnClickListener;
import android.widget.AdapterView;
import android.widget.AdapterView.OnItemClickListener;
import android.widget.AutoCompleteTextView;
import android.widget.Button;
import android.widget.TextView;

import com.tencent.urldatabase.SuggestionDBManage;

public class MainActivity extends Activity implements OnItemClickListener{
    
    private AutoCompleteTextView mAutoCompleteTextView;
    private Button mSaveButton;
    private SuggestionDBManage mUrlDBManage;

    /** Called when the activity is first created. */
    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.main);
        mUrlDBManage = new SuggestionDBManage(getApplicationContext());
        
        mAutoCompleteTextView = (AutoCompleteTextView)findViewById(R.id.autotextview);
        mAutoCompleteTextView.setText(""); 
        mAutoCompleteTextView.setOnItemClickListener(this);
        
        SuggestionAdapter adapter = new SuggestionAdapter(this, mUrlDBManage);
        mAutoCompleteTextView.setAdapter(adapter);
        
        mSaveButton = (Button)findViewById(R.id.savaurl);
        mSaveButton.setOnClickListener(new OnClickListener() {
            //这部分代码只是为了添加初始数据，只考虑http://开头的地址
            @Override
            public void onClick(View v) {
                String prefix = "http://";
                String urlString = mAutoCompleteTextView.getText().toString();
                if (!urlString.startsWith(prefix)) {
                    mUrlDBManage.insert(new SuggestionItem(prefix + urlString, "unknow"));
                }
                else {
                    mUrlDBManage.insert(new SuggestionItem(urlString, "unknow"));
                }
            }
        });
    }

    @Override
    protected void onDestroy() {
        mUrlDBManage.close();
        super.onDestroy();
    }

 
    @Override
    public void onItemClick(AdapterView<?> parent, View view, int position,
            long id) {
        TextView urlteTextView = (TextView)view.findViewById(R.id.title);
        mAutoCompleteTextView.setText(urlteTextView.getText());
    }

}
{% endhighlight %}

main.xml

{% highlight java %}
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="fill_parent"
    android:layout_height="fill_parent"
    android:orientation="horizontal" >

    <AutoCompleteTextView
        android:id="@+id/autotextview"
        android:layout_width="0dp"
        android:layout_weight="1"
        android:layout_height="wrap_content"
        android:singleLine="true" />
    
    <Button 
        android:id="@+id/savaurl"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="save"/>
</LinearLayout>
{% endhighlight %}

simple_list_item_for_autocomplete.xml

{% highlight java %}
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="horizontal" >
    
    <TextView 
        android:id="@+id/url"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_weight="1"
        android:textSize="16sp"/>
    
    <TextView  
        android:id="@+id/title"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_weight="1"
        android:gravity="right"
        android:textSize="16sp"/>

</LinearLayout>
{% endhighlight %}
