---
layout: post
title:  "从0开始搭建MVP+ViewModel框架的android应用01---MVP+ViewModel诞生记"
desc: "从0开始搭建应用系列，集MVP和MVVM两家之长，即将View和业务逻辑解耦了，也简化了Presenter的复杂性。"
keywords: "android,MVP"
date: 2017-03-05
categories: [Android]
tags: [android, MVP, ViewModel]
---
### 对比MVC/MVP/MVVM
* MVC：经典的模式，model，view，controller，比较好理解，但是有些缺点，承担View角色的模块包含了过多的业务逻辑
* MVP：衍生于MVC，虽然View和业务解耦了，但是Presenter承担了太多任务
* MVVM：采用DataBinding，数据的渲染自动反映在ViewModel上，同时也可以通过ViewModel获取数据，但是业务处理堆在一块。

基于此，想让处理不同事情的模块独立起来，通过接口解耦，Presenter只负责remote data／native data的获取，然后由ViewModel把数据跟View绑定起来，汲取MVP和MVVM所长，于是有了MVPVM。

补充：对于架构模式的搭建，没法说哪种更好，只有哪种更合适，不要局限于架构，结合业务特性，现有的人力和任务时间（采用架构代码量一般会增加，但是结构会清晰），选取适合的框架。

### MVPVM
参照MVP模式，采取contract接口隔离的方式，分离出View，Presenter，ViewWapper；Activity／Fragment实现View接口，提供Context下的API调用等，Presenter实现当前模块的业务数据获取，ViewWrapper则负责把数据传递给ViewModel实现数据绑定，以及View传递的Event处理。如下图：

<img src="/static/img/blog/mvpvm/mvp_vm_structure.jpeg" width="50%">

### 抽离出Base类。
* BaseView ——— 抽象出来的需要Context环境的调用接口 
* Presenter ——— 为实现架构搭建，由BasePresenter实现接口
* ViewWrapper ——— 为实现架构搭建，由BaseViewWrapper实现接口
* BaseActivity ——— 抽象出来的Base，即可由MvpVmActivity继承，也可由业务Activity继承
* BasePresenter ——— 抽象出来的MvpVm架构的Presenter基类
* BaseViewWrapper ——— 抽象出来的MvpVm架构的ViewWrapper基类
* MvpVmActivity ——— 抽象出来的MvpVm架构的Activity基类
* MvpVmFragment ——— 抽象出来的MvpVm架构的Fragment基类
* DemoActivity


#### BaseView
```java
public interface BaseView {

public void setTitle(int titleId);

public void setTitle(String title);

public void showToast(int resId);

public void showToast(String msg);

public void showWaitDialog(int resId);

public void showWaitDialog(String message);

....
}
```

#### Presenter
```java
public interface Presenter<V, VW> {
    void attachView(V view);
    
    void setViewWrapper(VW viewWrapper);
    
    void detachView();
}
```

#### ViewWrapper
```java
public interface ViewWrapper<V, D> {
    void attachView(V view);
    
    void detachView();
    
    void setBinding(D dataBinding);
    
    void onBind();
}
```
#### BaseActivity
```java
public class BaseActivity<D extends ViewDataBinding> extends AppCompatActivity implements BaseView {

    ...
    
    protected D dataBinding;
    protected BaseActivity activity;
    private BaseActivityDataBinding baseBinding;

    protected <D extends ViewDataBinding> D generateDataBinding(@LayoutRes int layoutResID) {
        D binding;
        if (hasToolBar()) {
            baseBinding = DataBindingUtil.setContentView(this, R.layout.activity_base);
            
            ...
            
            LayoutInflater inflater = (LayoutInflater) getSystemService(Context.LAYOUT_INFLATER_SERVICE);
            binding = DataBindingUtil.inflate(inflater, layoutResID, baseBinding.contentLayout, true);
        } else {
            binding = DataBindingUtil.setContentView(this, layoutResID);
        }
        activity = this;
        
        ...
        
        return binding;
    }
}
```
generateDataBinding替换一般情况下调用的setContentView(int layoutId)，在业务Avtivity中第一步调用dataBinding = generateDataBinding(R.layout.xxx)即返回对应layout的DataBinding，用法后面demo中将会介绍。

#### BasePresenter
```java
public class BasePresenter<V, VW> implements Presenter<V, VW> {

    public V view;
    protected VW viewWrapper;
    ...
    
    @Override
    public void attachView(V view) {
        this.view = view;
    }
    
    @Override
    public void setViewWrapper(VW viewWrapper) {
        this.viewWrapper = viewWrapper;
    }
    
    @Override
    public void detachView() {
        view = null;
        viewWrapper = null;
        ...
    }
    
    ...
	
}
```

#### BaseViewWrapper
```java
public class BaseViewWrapper<V, D extends ViewDataBinding> implements ViewWrapper<V, D> {
    protected V view;
    protected D dataBinding;
        
    @Override
    public void attachView(V view) {
        this.view = view;
    }
        
    @Override
    public void detachView() {
        view = null;
        if (dataBinding != null) {
            dataBinding.unbind();
            dataBinding = null;
        }
    }
        
    @Override
    public void setBinding(D dataBinding) {
        this.dataBinding = dataBinding;
        onBind();
    }

    @Override
    public void onBind() {
        
    }
    
}
```
onBind方法由业务ViewWrapper实现，具体操作为DataBinding的数据绑定，以及listener等事件的设置，会在后续博客中详细介绍。

#### MvpVmActivity
```java
public abstract class MvpVmActivity<P extends BasePresenter, VW extends BaseViewWrapper, D extends ViewDataBinding>
extends BaseActivity<D> {

    protected P presenter;
    protected VW viewWrapper;
    
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        presenter = createPresenter();
        viewWrapper = createViewWrapper();
        if (presenter != null && viewWrapper != null) {
            presenter.setViewWrapper(viewWrapper);
        }
    }
    
    protected abstract P createPresenter();
    
    protected abstract VW createViewWrapper();
    
    @Override
    protected void onDestroy() {
        if (presenter != null) {
            presenter.detachView();
        }
        if (viewWrapper != null) {
            viewWrapper.detachView();
        }
        if (dataBinding != null) {
            dataBinding.unbind();
            dataBinding = null;
        }
        super.onDestroy();
    }
}
```
createPresenter和createViewWrapper由业务Activity继承实现，实例化对应业务的Presenter和ViewWrapper，后续博客会详细介绍。

#### MvpVmFragment
```java
public abstract class MvpVmFragment<P extends BasePresenter, VW extends BaseViewWrapper, D extends ViewDataBinding>
    extends BaseFragment<D> {
    
    protected P presenter;
    protected VW viewWrapper;
    
    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        presenter = createPresenter();
        viewWrapper = createWrapper();
        if (presenter != null && viewWrapper != null) {
            presenter.setViewWrapper(viewWrapper);
        }
    }
    
    protected abstract P createPresenter();
    
    protected abstract VW createWrapper();
    
    @Override
    public void onDestroyView() {
        if (presenter != null) {
            presenter.detachView();
        }
        if (viewWrapper != null) {
            viewWrapper.detachView();
        }
        if (dataBinding != null) {
            dataBinding.unbind();
            dataBinding = null;
        }
        super.onDestroyView();
    }
}
```
createPresenter和createViewWrapper由业务Activity继承实现，实例化对应业务的Presenter和ViewWrapper，但是DataBinding实例的获取与在Activity中略有不同，是在onCreateView中获取的，后续博客会详细介绍。

#### DemoActivity
```java
public class MvpVmDemoActivity extends MvpVmActivity<MvpVmDemoPresenter, MvpVmDemoViewWrapper, MvpVmDemoDataBinding>
    implements MvpVmDemoContract.View {
    
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        dataBinding = generateDataBinding(R.layout.activity_mvpvm_demo);
        if (viewWrapper != null) {
            viewWrapper.setBinding(dataBinding);
        }
        presenter.fetchData();
    }
    
    @Override
    protected MvpVmDemoPresenter createPresenter() {
        return new MvpVmDemoPresenter(this);
    }
    
    @Override
    protected MvpVmDemoViewWrapper createViewWrapper() {
        return new MvpVmDemoViewWrapper(this);
    }
    
    ...
    
}
```

从代码中可以看到，在onCreate第一步就是生成对应的DataBinding实例（关于DataBinding会在后续博客中专门讲一次），然后通过setBinding设置给ViewWrapper，实现数据绑定关系，在createPresenter和createViewWrapper中实例化对应业务的presenter和viewWrapper。


截至到此，MVPVM的Base框架就大体搭起来了，后续还会更新基于此框架的业务细节实现以及基础工具的实现（如Adapter、RecyclerView、DataBinding的高级用法以及网络请求框架Retrofit二次封装等）

才疏学浅，板砖轻拍




