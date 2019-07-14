---
title: mvp todo
---

本片博客，简单介绍个人对mvp设计模式的理解（如有错误，请大家指教），源码地址 [google-todo-mvp](https://github.com/googlesamples/android-architecture/tree/todo-mvp) 。

理解：mvp通过，Interface.View Interface.Presenter 把本在fragment中的网络请求，放到presenter中完成。在fragment的生命周期，刷新、点击等事件中，执行presenter.method(){}; 而在Presenter中 method(){fragmet.updateView(data)}; 完成数据的展示，并规范fragment中的方法，使功能都有对应的方法，避免一个方法中出现多功能代码过长。

<!-- more -->

## mvp 结构

### TasksActivity

创建TasksPresenter，使TasksPresenter拿到tasksFragment对象。

``` bash
// Create the presenter
mTasksPresenter = new TasksPresenter(
      Injection.provideTasksRepository(getApplicationContext()), tasksFragment);
```


### TasksFragment

通过setPresenter（presenter）使，fragment获取到mPresenter对象，并在对应的事件中执行mPresenter的方法。

``` bash
public class TasksFragment extends Fragment implements TasksContract.View {
    @Override
    public void onResume() {
        super.onResume();
        mPresenter.start();
    }

    @Override
    public void setPresenter(@NonNull TasksContract.Presenter presenter) {
        mPresenter = checkNotNull(presenter);
    }
```

### TasksPresenter

在构造方法做，调用fragment的mTasksView.setPresenter(this);使fragment得到mPresenter对象。

``` bash
public TasksPresenter(@NonNull TasksRepository tasksRepository, @NonNull TasksContract.View tasksView) {
       mTasksRepository = checkNotNull(tasksRepository, "tasksRepository cannot be null");
       mTasksView = checkNotNull(tasksView, "tasksView cannot be null!");

       mTasksView.setPresenter(this);
   }
```

### TasksRepository

存储Task数据的，从远程获取，并进行Cache、local的本地化存储，local存储使用了SQLiteOpenHelper。

``` bash
// Prevent direct instantiation.
private TasksRepository(@NonNull TasksDataSource tasksRemoteDataSource,
                        @NonNull TasksDataSource tasksLocalDataSource) {
    mTasksRemoteDataSource = checkNotNull(tasksRemoteDataSource);
    mTasksLocalDataSource = checkNotNull(tasksLocalDataSource);
}
```

## mvp 调用

### 在TasksFragment中执行刷新

在事件onRefresh()中执行，mPresenter.loadTasks(false);

```
swipeRefreshLayout.setOnRefreshListener(new SwipeRefreshLayout.OnRefreshListener() {
            @Override
            public void onRefresh() {
                mPresenter.loadTasks(false);
            }
        });
```

### TasksPresenter中执行loadTasks(boolean forceUpdate)

Presenter拉取数据Tasks，最终mTasksView.showTasks(tasks);在fragment中刷新页面。

```
  @Override
   public void loadTasks(boolean forceUpdate) {
       // Simplification for sample: a network reload will be forced on first load.
       loadTasks(forceUpdate || mFirstLoad, true);
       mFirstLoad = false;
   }


   private void loadTasks(boolean forceUpdate, final boolean showLoadingUI) {
        if (showLoadingUI) {
            mTasksView.setLoadingIndicator(true);
        }
        if (forceUpdate) {
            mTasksRepository.refreshTasks();
        }

        mTasksRepository.getTasks(new TasksDataSource.LoadTasksCallback() {
            @Override
            public void onTasksLoaded(List<Task> tasks) {
              processTasks(tasks);
            }
          }
        }

    private void processTasks(List<Task> tasks) {
        if (tasks.isEmpty()) {
            // Show a message indicating there are no tasks for that filter type.
            processEmptyTasks();
        } else {
            // Show the list of tasks
            mTasksView.showTasks(tasks);
            // Set the filter label's text.
            showFilterLabel();
        }
    }
```

### 在TasksFragment中showTasks(List<Task> tasks)

最终在Interface.View中定义的showTasks(List<Task> tasks)，完成页面数据的刷新，实现整个refresh方法。

```
    @Override
    public void showTasks(List<Task> tasks) {
        mListAdapter.replaceData(tasks);

        mTasksView.setVisibility(View.VISIBLE);
        mNoTasksView.setVisibility(View.GONE);
    }
```

## mvp 的实战使用

[Google iosched](https://github.com/google/iosched) 一篇实际运用mvp的项目，封装优化了mvp需要创建多Interface的情况，并综合运用。**提醒：这个项目登录需要在有google账号的环境下，建议使用Android Studio的模拟器，并在Chrome登录你的Gmail。**
