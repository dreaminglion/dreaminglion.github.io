---
layout: post
title:  "iosched"
date:   2016-10-31 16:54:00 +0800
categories: iosched
---

google open source app iosched2015

<!-- more -->

 **welcome pages**

![](https://raw.githubusercontent.com/dreaminglion/dreaminglion.github.io/master/images/tos_frag.png)



## abstract WelcomeFragment

### 获取activity对象，并在onDetach时释放

```
protected Activity mActivity;

@Override
public void onAttach(Activity activity) {
        super.onAttach(activity);
        LOGD(TAG, "Attaching to activity");
        mActivity = activity;
}

@Override
public void onDetach() {
        super.onDetach();
        mActivity = null;
}
```

### 规范抽象方法，控制每个fragment对应的‘确定’、‘取消’按钮的文本以及点击事件。（由每个子fragment，来控制寄主activity button的文本和点击事件）

- protected abstract String getPositiveText();
- protected abstract View.OnClickListener getPositiveListener();
- protected abstract String getNegativeText();
- protected abstract View.OnClickListener getNegativeListener();
- protected abstract class WelcomeFragmentOnClickListener implements View.OnClickListener(规范‘确定’、‘取消’按钮的点击事件)

### interface WelcomeFragmentContainer (由WelcomeActivity实现，通过接口activity与fragment进行交互。fragment来调用activity中的代码，获取activity中的button对象)

- public Button getPositiveButton();
- public void setPositiveButtonEnabled(Boolean enabled);
- public Button getNegativeButton();
- public void setNegativeButtonEnabled(Boolean enabled);


## TosFragment extends WelcomeFragment

### 实现父类的abstract方法，获取button文本、clickListener

### implements WelcomeActivity.WelcomeActivityContent

- 实现shouldDisplay下次本引导fragment是否需要再次显示
- 多个引导fragment都实现通一个interface方便实用集合管理


## WelcomeActivity

**//实现WelcomeFragment的接口提供button对象
public class WelcomeActivity extends AppCompatActivity implements WelcomeFragment.WelcomeFragmentContainer**

### welcomeFragment都实现接口WelcomeActivityContent，方便集合管理以及获取当前应当展示的引导welcome fragment。


```
WelcomeActivityContent mContentFragment;

mContentFragment = getCurrentFragment(this);

FragmentTransaction fragmentTransaction = getFragmentManager().beginTransaction();
        // 实现接口的fragment，直接强转为fragment
        fragmentTransaction.add(R.id.welcome_content, (Fragment) mContentFragment);
        fragmentTransaction.commit();

private static WelcomeActivityContent getCurrentFragment(Context context) {
        List<WelcomeActivityContent> welcomeActivityContents = getWelcomeFragments();

for (WelcomeActivityContent fragment : welcomeActivityContents) {
            // 通过shared preference来确定welcome fragment的协议用户是否同意
            if (fragment.shouldDisplay(context)) {
                return fragment;
            }
        }
        return null;
    }        

   /**
     * Get all WelcomeFragments for the WelcomeActivity.
     *
     * @return the List of WelcomeFragments.
     */
private static List<WelcomeActivityContent> getWelcomeFragments() {
        return new ArrayList<WelcomeActivityContent>(Arrays.asList(
            new TosFragment(),
            new ConductFragment(),
            new AttendingFragment(),
            new AccountFragment()
        ));
}
```
