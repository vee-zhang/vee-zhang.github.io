# View的前前后后


## setContentView

```java
@Override
public void setContentView(View v) {
    ensureSubDecor();
    ViewGroup contentParent = mSubDecor.findViewById(android.R.id.content);
    contentParent.removeAllViews();
    contentParent.addView(v);
    mAppCompatWindowCallback.getWrapped().onContentChanged();
}

@Override
public void setContentView(int resId) {
    ensureSubDecor();
    ViewGroup contentParent = mSubDecor.findViewById(android.R.id.content);
    contentParent.removeAllViews();
    LayoutInflater.from(mContext).inflate(resId, contentParent);
    mAppCompatWindowCallback.getWrapped().onContentChanged();
}

@Override
public void setContentView(View v, ViewGroup.LayoutParams lp) {
    ensureSubDecor();
    ViewGroup contentParent = mSubDecor.findViewById(android.R.id.content);
    contentParent.removeAllViews();
    contentParent.addView(v, lp);
    mAppCompatWindowCallback.getWrapped().onContentChanged();
}
```


