---
title: Android ListView源码
tag: Android

---

<meta name="referrer" content="no-referrer" />



# ListView源码

## RecycleBin缓存

```java
class RecycleBin {
    private RecyclerListener mRecyclerListener;

    /**
     * The position of the first view stored in mActiveViews.
     */
    private int mFirstActivePosition;

    /**
     * Views that were on screen at the start of layout. This array is populated at the start of
     * layout, and at the end of layout all view in mActiveViews are moved to mScrapViews.
     * Views in mActiveViews represent a contiguous range of Views, with position of the first
     * view store in mFirstActivePosition.
     */
    private View[] mActiveViews = new View[0];

    /**
     * Unsorted views that can be used by the adapter as a convert view.
     */
    private ArrayList<View>[] mScrapViews;

    private int mViewTypeCount;

    private ArrayList<View> mCurrentScrap;

    private ArrayList<View> mSkippedScrap;

    private SparseArray<View> mTransientStateViews;
    private LongSparseArray<View> mTransientStateViewsById;

    public void setViewTypeCount(int viewTypeCount) {
        if (viewTypeCount < 1) {
            throw new IllegalArgumentException("Can't have a viewTypeCount < 1");
        }
        //noinspection unchecked
        ArrayList<View>[] scrapViews = new ArrayList[viewTypeCount];
        for (int i = 0; i < viewTypeCount; i++) {
            scrapViews[i] = new ArrayList<View>();
        }
        mViewTypeCount = viewTypeCount;
        mCurrentScrap = scrapViews[0];
        mScrapViews = scrapViews;
    }
}
```

- setViewtypeCount()
    我们都知道Adapter当中可以重写一个getViewTypeCount()来表示ListView中有几种类型的数据项，而setViewTypeCount()方法的作用就是为每种类型的数据项都单独启用一个RecycleBin缓存机制。在这个方法中，会根据你设置有多少种类型的数据，建立多少个ArrayList的废弃池；mScrapViews就是一个ArrayList\<View>类型的数组，它的大小就是viewTypeCount，根据多少种类型，分别存放View；mCurrentScrap就是默认的第一种数据类型，也就是我们常用的