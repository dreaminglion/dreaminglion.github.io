---
title: mvp iosched
---

mvp设计模式的实际运用，只用实现一个fragment Presenter，复杂逻辑使用fragment Presenter，简单页面的activity、fragment不使用presenter，源码地址 [google-iosched-mvp](https://github.com/dreaminglion/iosched-reader) 。

<!-- more -->

- M：ExploreModel 页面展示对应的数据集合，在Model内部完成数据的请求，缓存等数据方面的功能。
- V：ExploreIOFragment 页面展示的fragment，完成对M的展示。
- P：PresenterFragmentImpl 继承fragment的控制器，有对应的生命周期，在P中调用M的请求，V的数据展示。

Event：给listview item绑定数据、click事件。

1. 实现接口 ExploreIOFragment implements CollectionViewCallbacks
2. 赋值接口对象 mCollectionView.setCollectionAdapter(this);
3. 实现接口方法，bind数据、事件 @Override public void bindCollectionItemView(Context context, View view, int groupId,int indexInGroup, int dataIndex,Object tag)
4. 控件设置数据 titleView.setText(sessionData.getSessionName());
5. 控件设置事件 startButton.setOnClickListener(messageData.getStartButtonClickListener());

### BaseActivity

在presenter对象中，绑定mode和view。

```bash
private void setUpPresenter(PresenterFragmentImpl presenter, FragmentManager fragmentManager,
                                int updatableViewResId, Model model, QueryEnum[] queries,
                                UserActionEnum[] actions) {
        UpdatableView ui = (UpdatableView) fragmentManager.findFragmentById(
                updatableViewResId);
        presenter.setModel(model);
        presenter.setUpdatableView(ui);
        presenter.setInitialQueriesToLoad(queries);
        presenter.setValidUserActions(actions);
}
```

### ExploreModel

在mode中，通过Uri获取网络数据。

```bash
@Override
    public Loader<Cursor> createCursorLoader(int loaderId, Uri uri, @Nullable Bundle args) {
        CursorLoader loader = null;
        loader = getCursorLoaderInstance(mContext, uri,
                    ExploreQueryEnum.SESSIONS.getProjection(), null, null,
                    ScheduleContract.Sessions.SORT_BY_TYPE_THEN_TIME);
        return loader;
    }
```

### ExploreIOFragment

public class ExploreIOFragment extends Fragment implements UpdatableView<exploremodel>,CollectionViewCallbacks
实现UpdatableView接口，完成数据的展示。实现CollectionViewCallbacks接口，实现listview item的数据绑定、点击事件。

```bash
  @Override
  public void displayData(ExploreModel model, QueryEnum query) {
      // Only display data when the tag metadata is available.
      if (model.getTagTitles() != null) {
          updateCollectionView(model);
      }
  }

   @Override
   public void bindCollectionItemView(Context context, View view, int groupId,
           int indexInGroup, int dataIndex, Object tag) {
        if (!TextUtils.isEmpty(sessionData.getDetails())) {
                descriptionView.setText(sessionData.getDetails());
        }
        if (messageData.getStartButtonClickListener() != null) {
                startButton.setOnClickListener(messageData.getStartButtonClickListener());
        }
  }
```

### PresenterFragmentImpl

presenter实现fragment的生命周期，使用mode，完成view的展示。

0. PresenterFragmentImpl 的 onActivityCreated 调用 manager.initLoader(mInitialQueriesToLoad[i].getId(), null, this); 会触发 onCreateLoader 开始异步获取数据。
1. 获取url
2. 创建cursor游标的key,用来存对应的值
3. 用cursor去请求数据
4. 数据请求完成
5. 读取cursor请求到的数据
6. 更新页面展示 - over

```bash
   @Override
   public Loader<Cursor> onCreateLoader(int id, Bundle args) {
       Loader<Cursor> cursorLoader = createLoader(id, args);
       //3.用cursor去请求数据
       mLoaderIdlingResource.onLoaderStarted(cursorLoader);
       return cursorLoader;
   }

   @VisibleForTesting
   protected Loader<Cursor> createLoader(int id, Bundle args) {
       //1.获取url
       Uri uri = mUpdatableView.getDataUri(QueryEnumHelper.getQueryForId(id, mModel.getQueries()));
       LOGD(TAG,"beast : " + id + " - " + uri);
       //2.创建cursor游标的key,用来存对应的值
       return mModel.createCursorLoader(id, uri, args);
   }

   @Override
   public void onLoadFinished(Loader<Cursor> loader, Cursor data) {
       //4.数据请求完成
       processData(loader, data);
       mLoaderIdlingResource.onLoaderFinished(loader);
   }

   @VisibleForTesting
   protected void processData(Loader<Cursor> loader, Cursor data) {
       QueryEnum query = QueryEnumHelper.getQueryForId(loader.getId(), mModel.getQueries());
       //5.读取cursor请求到的数据
       boolean successfulDataRead = mModel.readDataFromCursor(data, query);
       if (successfulDataRead) {
           //6.更新页面展示 - over
           mUpdatableView.displayData(mModel, query);
       } else {
           mUpdatableView.displayErrorMessage(query);
       }
   }
```
