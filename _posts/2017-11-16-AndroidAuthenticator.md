---
layout: post
title: On Android Authenticator
---

本文简单介绍下Android account及其相关的类， 用户登录account的主要过程.

## Android account

Android 中的account即用户账号， 与我们通常理解的网络账号无异， 比如Gmail account, facebook account.对于某类服务比如Gmail， 用户需要自己的账号去登录， 保存自己的数据。

## 主要概念

- credentials 用户证书， 比如用户名，密码

- Auth-token

  Authentication Token -  服务器提供的临时访问token.  用户需要在登录服务器之后获取这个token， 在之后发送给服务器的request中需要附加这个token.

## 用户登录过程涉及到的Android类

- AccountManager

  Android framework层的类

- AccountAuthenticator

  app需要实现的类， 用于从服务器获取token等. 会被AccountManager调用.

- AccountAuthenticatorActivity

  与用户的交互activity， 比如用于提示用户输入用户名、密码进行登录.

## App登录过程
![classes relation](/images/accountmanager.png "classes relation")

- app首次登录， 调用AccountManager获取 authToken.
- AccountManager查询 AccountAuthenticator是否可以获得authToken
- 因为首次登录还没有获取过authToken, 所以启动AccountAuthenticatorActivity, 提示用户登录账号
- 用户登录， 获得server返回的 authToken
- AccountManager存储authToken供将来使用
- app得到了authToken

## 如何实现Authenticator

- 在AndroidManifest.xml中声明service

  ```
  <service android:name=".authentication.TestAuthenticatorService">
      <intent-filter>
          <action android:name="android.accounts.AccountAuthenticator" />
      </intent-filter>
      <meta-data android:name="android.accounts.AccountAuthenticator"
                 android:resource="@xml/authenticator" />
  </service>
  ```

- 定义service

  ```
  public class TestAuthenticatorService extends Service {
      @Override
      public IBinder onBind(Intent intent) {
          TestAuthenticator authenticator = new TestAuthenticator(this);
          return authenticator.getIBinder();
      }
  }
  ```

- 实现Authenticator

  继承 [*AbstractAccountAuthenticator*](http://developer.android.com/reference/android/accounts/AbstractAccountAuthenticator.html) 并实现该类的关键方法如addAccount(), getAuthToken()等.

- 创建交互Activity

  ```
  private void finishLogin(Intent intent) {
      String accountName = intent.getStringExtra(AccountManager.KEY_ACCOUNT_NAME);
      String accountPassword = intent.getStringExtra(PARAM_USER_PASS);
      final Account account = new Account(accountName, intent.getStringExtra(AccountManager.KEY_ACCOUNT_TYPE));
      if (getIntent().getBooleanExtra(ARG_IS_ADDING_NEW_ACCOUNT, false)) {
          String authtoken = intent.getStringExtra(AccountManager.KEY_AUTHTOKEN);
          String authtokenType = mAuthTokenType;
          // Creating the account on the device and setting the auth token we got
          // (Not setting the auth token will cause another call to the server to authenticate the user)
          mAccountManager.addAccountExplicitly(account, accountPassword, null);
          mAccountManager.setAuthToken(account, authtokenType, authtoken);
      } else {
          mAccountManager.setPassword(account, accountPassword);
      }
      setAccountAuthenticatorResult(intent.getExtras());
      setResult(RESULT_OK, intent);
      finish();
  }
  ```

  ​

## Reference

http://blog.udinic.com/2013/04/24/write-your-own-android-authenticator



