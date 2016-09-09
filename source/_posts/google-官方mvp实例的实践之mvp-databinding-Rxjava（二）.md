---
title: google 官方mvp实例的实践之mvp-databinding-Rxjava（二）
date: 2016-09-06 14:20:21
tags: Android
categories: Android
---

这篇文章主要是承接上一部分，给出我实现的mvp的主要实现代码；
没有看过上一篇文章的建议看一下上一篇[google 官方mvp实例的实践之mvp-databinding-Rxjava（一）](https://jiangchunyu.github.io/2016/09/06/google-%E5%AE%98%E6%96%B9mvp%E5%AE%9E%E4%BE%8B%E7%9A%84%E5%AE%9E%E8%B7%B5%E4%B9%8Bmvp-databinding-Rxjava-%E4%B8%80/)
闲话少许，继续放码；
[demo地址](https://github.com/jiangchunyu/googlemvp)
<!--more-->
```
public class LoginCotract {
    public interface View extends BaseView<Presenter>{
        void loginSuccess();
        void loginError(String error);

    }

    public interface Presenter extends BasePresenter{
        void login(String userName,String pwd);
        void loginSuccess();
        void loginError(String error);
    }
}
```
```
public class LoginActivity extends BaseActivity implements LoginCotract.View {

    private ActLoginBinding mBinding;
    private LoginPresenter mPresenter;
    private DigLoginWait mDigLoginWait;

    @Override
    protected int getLayoutId() {
        return R.layout.act_login;
    }

    @Override
    protected void onResume() {
        super.onResume();
        mPresenter.onResume(this);
        mDigLoginWait = new DigLoginWait(this);
    }

    @Override
    protected void onStop() {
        super.onStop();
        mPresenter.onStop();
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        mPresenter.onDestroy();
    }

    @Override
    protected void initBinding(ViewDataBinding binding) {
        mBinding = (ActLoginBinding) binding;
    }


    /**
     * 登陆按钮
     *
     * @param view
     */
    public void onLogin(View view) {
        //通过databinding 实现对数据双向绑定
        Log.e("jcy", "UserName  " + mBinding.getUserName() + "  Password " + mBinding.getPassword());
        mDigLoginWait.showDialog();
        mPresenter.login(mBinding.getUserName(), mBinding.getPassword());
    }

    /**
     * 登陆成功
     */
    @Override
    public void loginSuccess() {
        mDigLoginWait.closeDialog();
        Toast.makeText(LoginActivity.this, "登陆成功", Toast.LENGTH_SHORT).show();
    }

    /**
     * 登陆失败
     *
     * @param error 登陆失败原因
     */
    @Override
    public void loginError(String error) {
        mDigLoginWait.closeDialog();
        Toast.makeText(LoginActivity.this, error, Toast.LENGTH_SHORT).show();
    }

    /**
     * 初始化Presenter
     */
    public void initPresenter() {
        Log.e("jcy", "  initPresenter ");
        mPresenter = new LoginPresenter(this);
    }

    /**
     * 不如果是在Fragment中可以在初始化调用Fragment时传入
     *
     * @param presenter
     */
    @Override
    public void setPresenter(LoginCotract.Presenter presenter) {
    }
}

```


```
package com.jcy.googlemvp.presenter;

import android.content.Context;
import android.util.Log;

import com.jcy.googlemvp.contract.LoginCotract;
import com.jcy.googlemvp.model.LoginModel;

/**
 * @className: LoginPresenter
 * @desc:
 * @author: Jiangcy
 * @datetime: 2016/9/3
 */
public class LoginPresenter implements LoginCotract.Presenter {

    private LoginCotract.View view;
    private LoginModel mModel;


    public LoginPresenter(LoginCotract.View view) {
        this.view = view;
        view.setPresenter(this);
        mModel = new LoginModel(this);
    }

    @Override
    public void login(String userName, String pwd) {
        mModel.login(userName, pwd);

    }

    @Override
    public void loginSuccess() {
        view.loginSuccess();
    }

    @Override
    public void loginError(String error) {
        view.loginError(error);
    }


    @Override
    public void onResume(Context context) {
        Log.e("jcy", "  onResume ");
        mModel.onResume(context);
    }

    @Override
    public void onStop() {
    }

    @Override
    public void onDestroy() {
        mModel.onDestroy();
    }
}

```

```
package com.jcy.googlemvp.model;

import android.util.Log;

import com.jcy.googlemvp.base.BaseModel;
import com.jcy.googlemvp.data.RequestData;
import com.jcy.googlemvp.presenter.LoginPresenter;
import com.jcy.googlemvp.rxbus.EventType;
import com.jcy.googlemvp.utils.StringUtil;

import rx.Observable;
import rx.Subscriber;
import rx.android.schedulers.AndroidSchedulers;
import rx.functions.Action1;
import rx.schedulers.Schedulers;

/**
 * @desc:
 * @author: Jiangcy
 * @datetime: 2016/9/3
 */
public class LoginModel extends BaseModel<LoginPresenter>{
    private final int LOGIN_ERROR = 0x0010;
    private final int LOGIN_SUCCESS = 0x0011;
    private static final String TAG = "LoginModel";

    public LoginModel(LoginPresenter mPresenter) {
       super(mPresenter);
        initReciever(EventType.REC_LOGIN);
    }




    public void login(final String userName, final String pwd) {
        Log.d(TAG, "login: userName "+userName+"  isBlank "+StringUtil.isBlank(userName));
        Log.d(TAG, "login: pwd      "+pwd+"  isBlank "+StringUtil.isBlank(pwd));
        if (StringUtil.isBlank(userName) || StringUtil.isBlank(pwd)) {
            mPresenter.loginError("用户名或者密码为空，请输入正确的用户名或密码");
            return;
        }
        //模拟网络登陆环境
        Observable.create(new Observable.OnSubscribe<RequestData>() {
            @Override
            public void call(Subscriber<? super RequestData> subscriber) {
                try {
                    Thread.sleep(2 * 1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                RequestData requestData = new RequestData();
                if (userName.equals("one")) {
                    //用户名不存在
                    requestData.setCode(LOGIN_ERROR);
                    requestData.setInfo("用户名不存在，请先注册");

                } else if (userName.equals("admin") || pwd.equals("admin")) {
                    //登陆成功
                    requestData.setCode(LOGIN_SUCCESS);
                    requestData.setInfo("");
                } else {
                    //用户名或者密码不正确
                    requestData.setCode(LOGIN_ERROR);
                    requestData.setInfo("用户名或者密码不正确");
                }
                subscriber.onNext(requestData);
            }
        }).subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread()).subscribe(new Action1<RequestData>() {
            @Override
            public void call(RequestData requestData) {
                if(requestData.getCode()==LOGIN_ERROR){
                    mPresenter.loginError(requestData.getInfo());
                }else if(requestData.getCode()==LOGIN_SUCCESS){
                    mPresenter.loginSuccess();
                }
            }
        }, new Action1<Throwable>() {
            @Override
            public void call(Throwable throwable) {
                mPresenter.loginError("网络出现问题，请重新登陆"+throwable.toString());
            }
        });
    }

    @Override
    public void onMainEvent(String action, Object event) {

    }

    @Override
    public void onThreadEvent(String action, Object event) {

    }
}

```
代码中已经加入了详细的说明，这里就不再过多的赘述了；
[demo地址](https://github.com/jiangchunyu/googlemvp)
